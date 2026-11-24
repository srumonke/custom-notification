# Customising Harness approval notifications with Custom Notification Templates

A working demo that answers the question:

> Can we customise the notification sent by the approval step (beautify it), and add the
> **promoted version** to it?

Short answer: **yes, but not by editing the built-in notification.** The auto-generated
"Approval Required" / "Approved / Rejected" emails are produced by Harness and their frame
cannot be restyled. There are two supported routes:

| Route | FF needed | What you get |
|---|---|---|
| **1. Approval Message expressions** | none, GA | Your own text, including `<+pipeline.variables.promotedVersion>`, rendered *inside* both built-in approval notifications. Content only — no styling. |
| **2. Custom Notification Templates** | `PIPE_CUSTOM_NOTIFICATION_TEMPLATES` | You own the whole subject + HTML body. Full control of layout, colour, links, and which expressions appear. |

This repo implements route 2 (and keeps route 1 in the pipeline as well, in the
`approvalMessage` field, so you can see both side by side in one run).

---

## Contents

```
notif_template/
  KyribaApprovalRequestedEmail.yaml   amber  - approval requested
  KyribaApprovalApprovedEmail.yaml    green  - promotion approved
  KyribaApprovalRejectedEmail.yaml    red    - promotion rejected
pipeline/
  approval_notif_promoted_version_demo.yaml
```

---

## The core problem: there is no "approval approved/rejected" event

The complete pipeline notification event enum is:

```
AllEvents, PipelineStart, PipelineSuccess, PipelineFailed, PipelineEnd, PipelinePaused,
StageSuccess, StageFailed, StageStart, StepFailed, TriggerFailed, WaitingForUserAction,
PipelineResumed
```

No approval-outcome event exists. So the approval lifecycle is mapped onto events that do:

| Approval moment | Notification rule | Template |
|---|---|---|
| Approval requested | `WaitingForUserAction` | `KyribaApprovalRequestedEmail` |
| Approved | `StageSuccess` scoped to `forStages: [promotion_approval]` | `KyribaApprovalApprovedEmail` |
| Rejected | `StageFailed` scoped to `forStages: [promotion_approval]` | `KyribaApprovalRejectedEmail` |

A rejection **fails** the `HarnessApproval` step, which fails the stage — that is what makes the
`StageFailed` mapping reliable. Scoping both rules with `forStages` is essential, otherwise any
other stage's success/failure also fires the approval card.

## Attaching a template to a rule (undocumented YAML field)

The UI exposes this, but the YAML field is not in the docs. It lives on each notification rule:

```yaml
notificationRules:
  - identifier: promotion_approved
    pipelineEvents:
      - type: StageSuccess
        forStages:
          - promotion_approval
    template:
      templateRef: KyribaApprovalApprovedEmail
      versionLabel: v1
    notificationMethod:
      type: Email
      spec:
        recipients:
          - someone@example.com
    enabled: true
```

Confirmed against `harness_schema(resource_type='pipeline', path='definitions.NotificationRules')`.

## Template shape

```yaml
template:
  identifier: KyribaApprovalApprovedEmail
  versionLabel: v1
  type: Notification
  spec:
    body:
      type: STRING          # JSON | YAML | STRING | HTML
      content: |-
        subject: ...
        body: <div>...</div>
```

For Email, the content is a `subject:` / `body:` pair. `STRING` is the type Harness' own
built-in `DefaultEmail` template uses, so it is the safest choice.

Expressions available in the body:

- everything pipeline-scoped — `<+pipeline.variables.*>`, `<+pipeline.name>`,
  `<+pipeline.sequenceId>`, `<+pipeline.triggeredBy.email>`, `<+pipeline.execution.url>`,
  `<+project.identifier>`
- notification-scoped — `<+notification.eventType>`, `<+notification.variables.X>`,
  `<+notification.errorMessage>`, `<+notification.themeColor>`, `<+notification.startDate>`,
  `<+notification.endDate>`, `<+notification.duration>`, `<+notification.imageStatus>`,
  `<+notification.nodeStatus>`
- approval step outputs —
  `<+pipeline.stages.<stageId>.spec.execution.steps.<stepId>.output.approvalActivities[0].user.name>`
  and `.action` / `.comments`

---

## Three gotchas, all of which fail silently

In every case the template saves cleanly, governance passes, the execution succeeds — and the
email is simply wrong. These cost most of the build time and are the most useful part of this repo.

### 1. A space before `#` wipes out the entire body

The body content is parsed as YAML, where whitespace + `#` starts a comment. So:

```
border:1px solid #d0d7de
```

truncates the body at ` #d0d7de`. What is actually sent is
`<div style="font-family:...;border:1px solid` — an unterminated tag that renders **nothing**.
The mail arrives with a correct subject and a completely blank body.

**Fix:** never use hex colours. Use `rgb(208,215,222)`. All three templates here are hex-free.

### 2. Colon-space in the body breaks parsing the same way

`Promoted version: 2026.11.3-rc4` inside the body is read as a new YAML key.

**Fix:** no `: ` anywhere in the body. The fact tables use two `<td>` cells
(`<td>Promoted version</td><td>2026.11.3-rc4</td>`) instead of `Label: value`.

### 3. Ternaries on `<+notification.eventType>` silently take the wrong branch

This was tried first, as a single decision template:

```
<+<+notification.eventType>=='StageSuccess'?'APPROVED':'REJECTED'>
```

It always resolved to the false branch — an **approved** stage produced a **REJECTED** card.

**Fix:** don't branch at all. The rule's event already determines the outcome, so there is one
template per outcome with the decision hardcoded. Each template also carries a
`Raw event value` row printing unmodified `<+notification.eventType>`, so if you do need to
branch you can first observe what the value actually is in a real send.

---

## Installing the demo

1. Confirm the feature flag `PIPE_CUSTOM_NOTIFICATION_TEMPLATES` is enabled for the account.
2. Create the three templates — **Project Settings → Templates → New Template → Notification**,
   then paste each file from `notif_template/` in the YAML editor. Or via API/Terraform as
   project-level templates at `versionLabel: v1`.
3. Create the pipeline from `pipeline/approval_notif_promoted_version_demo.yaml`.
4. **Replace `REPLACE_ME@example.com`** in all three notification rules with real recipients,
   and change `approvers.userGroups` from `_project_all_users` if you want a narrower group.
5. The pipeline targets `orgIdentifier: default` / `projectIdentifier: Kyriba` — adjust for
   your scope. Both CI stages run on Harness Cloud (`runtime.type: Cloud`,
   `cloneCodebase: false`), so no delegate and no codebase connector is needed.

## Running it

1. Run the pipeline; keep the defaults (`payments-api`, `2026.11.3-rc4`, `PROD-EU1`).
2. **Resolve Promotion** echoes the promotion candidate.
3. **Promotion Approval** pauses → amber *"Promotion awaiting approval"* email with the promoted
   version in the subject, in the fact table, and in the body sentence. The built-in approval
   notification also fires, carrying the `approvalMessage` text — that is route 1 for comparison.
4. Approve → green *"Promotion APPROVED"* email, with approver name and comments.
   Reject → red *"Promotion REJECTED"* email.

## Adapting it

The only thing making this about "promoted version" is the pipeline variable
`promotedVersion`. In a real pipeline that would be fed from a trigger payload or an artifact
output rather than `<+input>.default(...)`; the templates don't change.
