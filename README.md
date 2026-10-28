# JSM Automation Recipes

![Jira Service Management](https://img.shields.io/badge/Jira%20Service%20Management-Automation-0052CC?style=flat&logo=jira&logoColor=white)

Ready-to-build **Automation for Jira** rules for **Jira Service Management Cloud**, written as recipes: trigger, conditions, actions and the smart values that make them work. Each one solves a recurring ITSM need: keeping requests moving, protecting SLAs and keeping related tickets in sync.

> Field names, request types, statuses and link types are examples. Map them to your project before building the rule, and test on a sandbox project first.

## Recipes

| # | Recipe | Main idea |
|---|---|---|
| 1 | [Back to support when the customer replies](#1-back-to-support-when-the-customer-replies) | Status loop on customer comments |
| 2 | [Pre-fill the description by request type](#2-pre-fill-the-description-by-request-type) | Templates for agents |
| 3 | [Warn before an SLA breach](#3-warn-before-an-sla-breach) | SLA threshold trigger |
| 4 | [Auto-close resolved requests](#4-auto-close-resolved-requests) | Scheduled clean-up |
| 5 | [Keep sub-tasks aligned with their parent](#5-keep-sub-tasks-aligned-with-their-parent) | Field copy with branches |
| 6 | [Update linked incidents when a problem is fixed](#6-update-linked-incidents-when-a-problem-is-fixed) | Cross-issue sync |
| 7 | [Detect a major incident](#7-detect-a-major-incident) | Lookup issues + threshold |

---

## 1. Back to support when the customer replies

**Goal:** a request waiting for the customer goes back to the agents' queue as soon as the customer answers.

| Step | Configuration |
|---|---|
| Trigger | **Issue commented** |
| Condition | **Issue fields condition**: Status = `Waiting for customer` |
| Condition | **Smart values condition**: `{{comment.internal}}` equals `false` |
| Condition | **Smart values condition**: `{{comment.author.accountId}}` equals `{{issue.reporter.accountId}}` |
| Action | **Transition issue** to `Waiting for support` |

**Why the two smart value checks:** without them, an internal note or an agent's own public reply would also move the ticket.

## 2. Pre-fill the description by request type

**Goal:** agents get a structured checklist in the ticket for each request type, without changing the customer form.

| Step | Configuration |
|---|---|
| Trigger | **Work item created** |
| Condition | **JQL condition**: `"Request Type" = "Access request"` |
| Action | **Edit issue** › Description (append) |

Template to append:

```text
----
*Agent checklist*
* Manager approval received: (/) (x)
* Account: {{issue.reporter.displayName}}
* Requested on: {{issue.created.format("dd/MM/yyyy")}}
* Due: {{now.plusBusinessDays(2).format("dd/MM/yyyy")}}
```

Duplicate the rule, or use an **If / else** block, for each request type.

## 3. Warn before an SLA breach

**Goal:** the assignee and the team lead know before the SLA is breached, not after.

| Step | Configuration |
|---|---|
| Trigger | **SLA threshold breached** › SLA `Time to resolution` › *will breach in the next 30 minutes* |
| Condition | **Issue fields condition**: Assignee is not empty |
| Action | **Send email** to Assignee + team lead |
| Action | **Add comment** (internal) |

Email body:

```text
{{issue.key}} - {{issue.summary}}
Time to resolution breaches in less than 30 minutes.
Priority: {{issue.priority.name}}
Open: {{issue.url}}
```

Add a second branch for unassigned tickets that notifies the queue owner instead.

## 4. Auto-close resolved requests

**Goal:** resolved requests close automatically when the customer does not react.

| Step | Configuration |
|---|---|
| Trigger | **Scheduled**, daily, with JQL: `project = SD AND status = Resolved AND updated <= -5d` |
| Action | **Add comment** (public): *This request has been closed automatically. Reply to reopen it.* |
| Action | **Transition issue** to `Closed` |

Keep the delay in line with the customer communication policy, and make sure a customer comment on a closed request reopens it (or creates a follow-up).

## 5. Keep sub-tasks aligned with their parent

**Goal:** when the parent's priority or due date changes, sub-tasks follow.

| Step | Configuration |
|---|---|
| Trigger | **Field value changed**: Priority, Due date |
| Branch | **For sub-tasks** |
| Action (in branch) | **Edit issue**: Priority › *Copy from Parent*, Due date › *Copy from Parent* |

The same pattern works for any field, including custom fields such as an Assets "Affected service".

## 6. Update linked incidents when a problem is fixed

**Goal:** when a problem is resolved, every incident it caused is informed and can be closed by the agents.

| Step | Configuration |
|---|---|
| Trigger | **Work item transitioned** to `Resolved` |
| Condition | **Issue fields condition**: Issue type = `Problem` |
| Branch | **For linked work items** › link type `causes` |
| Action (in branch) | **Add comment** (internal): *Root cause {{triggerIssue.key}} resolved: {{triggerIssue.resolution.name}}. Check with the customer and close.* |
| Action (in branch) | **Transition issue** to `Waiting for customer` (optional) |

`{{triggerIssue}}` refers to the problem that fired the rule, while `{{issue}}` inside the branch is each linked incident.

## 7. Detect a major incident

**Goal:** raise the alarm when several incidents hit the same service in a short time.

| Step | Configuration |
|---|---|
| Trigger | **Work item created** |
| Condition | **Issue fields condition**: Issue type = `Incident` |
| Action | **Lookup work items**: `project = SD AND issuetype = Incident AND "Affected services" = "{{issue.Affected services}}" AND created >= -30m` |
| Condition | **Smart values condition**: `{{lookupIssues.size}}` greater than `4` |
| Action | **Send message** (Slack / Teams) to the on-call channel |
| Action | **Add comment** (internal) listing `{{#lookupIssues}}{{key}} {{/}}` |

Tune the threshold and the window to the volume of the service desk to avoid false alarms.

---

## Good practices

- **One rule, one purpose.** Small rules are easier to debug than one rule with ten branches.
- **Prefer project-scoped rules.** Global and multi-project rules consume more executions.
- **Check the audit log** after each change: it shows every condition evaluated and why a rule stopped.
- **Guard against loops.** Rules that edit issues can trigger other rules: keep *Allow rule trigger* off unless it is needed.
- **Test smart values** with a temporary *Log action* before using them in emails or edits.

## Contributing

Have a recipe that saved you hours? Open an issue or a pull request.

## License

MIT
