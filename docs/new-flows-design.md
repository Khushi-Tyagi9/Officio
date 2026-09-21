# New flows: design for manual build in Power Automate

**Status: untested drafts.** These were written from the existing flows in `unpacked/Workflows/` and the app sources in `apps_src/`. None of it has been built or run, and the expressions have not been checked in the designer. Build in a dev environment first, and inside the LeaveManager solution so connection references and environment variables travel with it.

Conventions used below: action names follow the existing `Snake_Case` style. "Dataverse" = the Microsoft Dataverse connector (use the existing connection reference `contoso_DVReference`). "Outlook" = Office 365 Outlook (`contoso_OutReference`).

## Facts these designs depend on

| Fact | Where it comes from |
|---|---|
| Every app action (submit, approve, decline, recall, HR approve, capture) sets `contoso_lastactortimestamp = Now()` | `ConfirmationScreen.fx.yaml` l.51, `ApproveRejectScreen.fx.yaml` l.260/300, `HomeScreen.fx.yaml` l.674, `hrapprove/.../MainScreen.fx.yaml` l.121/133, `hrcapture/.../MainScreen.fx.yaml` l.89 |
| A new request is created directly as `Pending Manager Approval` | `ConfirmationScreen.fx.yaml` l.43 |
| The approver's address is stored as the **UPN** in `contoso_approveremail` | `LeaveApproverScreen.fx.yaml` l.273 |
| `contoso_leavemanagersetting` has **no HR email or admin email column** | `Entities/contoso_LeaveManagerSetting/Entity.xml` |
| The manager's manager is not stored anywhere | no such column; it must be looked up |
| Approved status strings: `Pending Manager Approval`, `Pending HR Approval`, `Pending HR Capture`, `Approved`, `Declined`, `Draft` | `ApprovalStatus` constants in `apps_src/requestleave/Src/App.fx.yaml` (l.79 onward); HR Approve uses the literals directly |

---

## Flow 1: Escalate Pending Approvals

**Purpose:** remind an approver after 2 days, escalate to the approver's manager after 4 days, never send either twice for the same submission.

### New Dataverse columns

| Table | Column (schema name) | Type | Default | Notes |
|---|---|---|---|---|
| `contoso_leaverequest` | `contoso_reminder_sent` | Two options (Yes/No) | No | Set to Yes after the 2-day reminder |
| `contoso_leaverequest` | `contoso_escalated` | Two options (Yes/No) | No | Set to Yes after the 4-day escalation |

Optional, if you want proof of when: `contoso_reminder_sent_on` and `contoso_escalated_on` (Date and time).

**Reset requirement (important).** If an employee recalls and resubmits, `contoso_lastactortimestamp` restarts, but the two flags stay Yes, so the second submission would never be reminded or escalated. Reset both to No on every submit. The simplest place is the app's submit `Patch` in `ConfirmationScreen.fx.yaml` (add the two columns with `false`). The alternative is an extra `Update_a_row` in the `Submitted` case of `LeaveRequestApproval`.

### Trigger

**Recurrence**, Interval 1, Frequency Day, At these hours `8`, At these minutes `0`, Time zone: your business time zone. (The existing solution assumes UTC+02:00.)

```json
"triggers": { "Recurrence": { "type": "Recurrence",
  "recurrence": { "frequency": "Day", "interval": 1,
    "timeZone": "<your time zone>", "schedule": { "hours": ["8"], "minutes": [0] } } } }
```

### Actions in order

| # | Action name | Connector | What it does / expression |
|---|---|---|---|
| 1 | `Compose_Reminder_Cutoff` | Data operation: Compose | `formatDateTime(addDays(utcNow(), -2), 'yyyy-MM-ddTHH:mm:ssZ')` |
| 2 | `Compose_Escalation_Cutoff` | Compose | `formatDateTime(addDays(utcNow(), -4), 'yyyy-MM-ddTHH:mm:ssZ')` |
| 3 | `Get_Leave_Manager_Settings_Row` | Dataverse: List rows | Table `contoso_leavemanagersettings`, Filter `contoso_name eq 'default'`, Top count 1 (same as the approval flow) |
| 4 | `Initialize_Power_App_URL` | Variables: Initialize (**top level**) | Name `LeaveManagerPowerAppURL`, String, value `@{first(outputs('Get_Leave_Manager_Settings_Row')?['body/value'])?['contoso_appurl']}&approvals=1` |
| 5 | `List_Pending_Manager_Requests` | Dataverse: List rows | Table `contoso_leaverequests`. Filter rows: `contoso_approvalstatus eq 'Pending Manager Approval' and contoso_lastactortimestamp le @{outputs('Compose_Reminder_Cutoff')} and (contoso_escalated eq false or contoso_escalated eq null)`. Turn **Pagination** on (threshold 5000). |
| 6 | `Apply_to_each_Request` | Control: Apply to each | Input `@outputs('List_Pending_Manager_Requests')?['body/value']`. Concurrency control on, degree 1 (keeps run order predictable and avoids mail bursts) |

Inside `Apply_to_each_Request`:

| # | Action name | Connector | What it does / expression |
|---|---|---|---|
| 6.1 | `Get_Created_By_User` | Dataverse: Get a row by ID | Table `systemusers`, Row ID `@items('Apply_to_each_Request')?['_createdby_value']` (same lookup the approval flow uses for the employee) |
| 6.2 | `Get_Leave_Type` | Dataverse: Get a row by ID | Table `contoso_leavetypes`, Row ID `@items('Apply_to_each_Request')?['_contoso_leavetype_value']` |
| 6.3 | `Condition_Is_4_Days_Or_Older` | Control: Condition | `@lessOrEquals(ticks(items('Apply_to_each_Request')?['contoso_lastactortimestamp']), ticks(outputs('Compose_Escalation_Cutoff')))` is equal to `true` |

**If yes (4+ days: escalate):**

| # | Action name | Connector | What it does / expression |
|---|---|---|---|
| Y1 | `Get_Approver_Manager` | Office 365 Users: Get manager (V2) | User (UPN): `@items('Apply_to_each_Request')?['contoso_approveremail']`. Works because the app stores the UPN. |
| Y2 | `Condition_Manager_Found` | Control: Condition | `@not(empty(outputs('Get_Approver_Manager')?['body/mail']))` is equal to `true` |
| Y2-yes | `Send_escalation_email` | Outlook: Send an email (V2) | To `@outputs('Get_Approver_Manager')?['body/mail']`, Cc `@items('Apply_to_each_Request')?['contoso_approveremail']`. Subject `Escalation: leave request waiting 4+ days for approval`. Body: requester `@{outputs('Get_Created_By_User')?['body/fullname']}`, leave type `@{outputs('Get_Leave_Type')?['body/contoso_name']}`, start/end/duration/comments from the row, submitted on `@{items('Apply_to_each_Request')?['contoso_lastactortimestamp']}`, link `@{variables('LeaveManagerPowerAppURL')}` |
| Y2-yes | `Update_row_Mark_Escalated` | Dataverse: Update a row | Table `contoso_leaverequests`, Row ID `@items('Apply_to_each_Request')?['contoso_leaverequestid']`, `contoso_escalated` = Yes, `contoso_reminder_sent` = Yes. **Do not** touch `contoso_lastactortimestamp`, or the age resets and `CreateHistoryRecord` fires. |
| Y2-no | `Send_escalation_no_manager_email` | Outlook | Approver has no manager in Office 365. Email the address stored in `contoso_hremail` (see Flow 2) so the request is not lost, then run the same `Update_row_Mark_Escalated` so it is not retried daily. |

**If no (2-3 days: remind):**

| # | Action name | Connector | What it does / expression |
|---|---|---|---|
| N1 | `Condition_Reminder_Not_Sent` | Control: Condition | `@not(equals(items('Apply_to_each_Request')?['contoso_reminder_sent'], true))` is equal to `true` |
| N1-yes | `Send_reminder_email` | Outlook: Send an email (V2) | To `@items('Apply_to_each_Request')?['contoso_approveremail']`, Cc `@outputs('Get_Created_By_User')?['body/internalemailaddress']`. Subject `Reminder: leave request waiting for your approval`. Same detail block plus the link. |
| N1-yes | `Update_row_Mark_Reminder_Sent` | Dataverse: Update a row | `contoso_reminder_sent` = Yes |
| N1-no | (nothing) | | Already reminded on an earlier day |

**Optional history row** after each send: Dataverse `Add_a_new_row` to `contoso_leaverequesthistories` with `contoso_LeaveRequest@odata.bind` = `contoso_leaverequests(<id>)`, `contoso_name` and `contoso_action` = `Reminder sent` or `Escalated`, `contoso_actor` = `System`. This mirrors `Add_Action_to_History_Table`.

### Design notes and weak points

- **"2+ days" is calendar time**, so a request submitted Friday is reminded on Sunday's run. Skipping weekends and the `contoso_holiday` table is possible but needs a working-day calculation; not included.
- **Send-then-flag order**: if the email succeeds and the update fails, tomorrow's run sends a duplicate reminder. Flipping the order instead risks a lost reminder. Sending first is the safer failure. Wrap with the Try/Catch from Flow 3.
- The approver's own status can change between the query and the send (approved a minute ago). Acceptable for a daily job; to be strict, re-read the row inside the loop and check the status again.
- Only *Pending Manager Approval* is covered. Requests stuck in *Pending HR Approval* or *Pending HR Capture* are not escalated.
- The 2 and 4 are hardcoded in the two Compose actions. To make them configurable add integer columns `contoso_reminderdays` and `contoso_escalationdays` to `contoso_leavemanagersetting` and read them from action 3.

---

## Flow 2: Notify HR of Pending Requests

**Purpose:** when a request reaches an HR stage, email HR. Today only the employee is told (`Send_email_after_HR_approval`, `Send_an_email_(V2)` in `LeaveRequestApproval`).

### New Dataverse columns

| Table | Column (schema name) | Type | Notes |
|---|---|---|---|
| `contoso_leavemanagersetting` | `contoso_hremail` | Single line of text, format Email | The HR mailbox or distribution list. Business recommended; add it to the `default` row after creating it. |

Optional: `contoso_hrapproveappurl` and `contoso_hrcaptureappurl` (single line text). HR Approve and HR Capture are separate apps, but the settings table only holds the main app URL (`contoso_appurl`). Without these, the email can only link to the main app.

**Role note:** `HR Leave Approvers` has no read on Leave Manager Setting. That does not matter to the flow (it runs as the flow owner's connection), but the owner needs read on the table.

### Trigger

Dataverse: **When a row is added, modified or deleted**, Change type **Modified**, Table `contoso_leaverequest`, Scope **Organization**, Select columns `contoso_approvalstatus`, Filter rows `contoso_approvalstatus eq 'Pending HR Approval' or contoso_approvalstatus eq 'Pending HR Capture'`. (The filter follows the pattern already used in `CreateRemoveMeetingRequest`, line 53.) Requests only move into these states by modification, never on create.

```json
"parameters": {
  "subscriptionRequest/message": 3,
  "subscriptionRequest/entityname": "contoso_leaverequest",
  "subscriptionRequest/scope": 4,
  "subscriptionRequest/filteringattributes": "contoso_approvalstatus",
  "subscriptionRequest/filterexpression": "contoso_approvalstatus eq 'Pending HR Approval' or contoso_approvalstatus eq 'Pending HR Capture'"
}
```

### Actions in order

| # | Action name | Connector | What it does / expression |
|---|---|---|---|
| 1 | `Get_Settings` | Dataverse: List rows | Table `contoso_leavemanagersettings`, Filter `contoso_name eq 'default'`, Top count 1 |
| 2 | `Condition_HR_Email_Configured` | Control: Condition | `@and(greater(length(outputs('Get_Settings')?['body/value']), 0), not(empty(first(outputs('Get_Settings')?['body/value'])?['contoso_hremail'])))` is equal to `true` |
| 2-no | `Terminate_Not_Configured` | Control: Terminate | Status **Failed**, code `HR_EMAIL_MISSING`, message `No default settings row, or contoso_hremail is empty`. This makes the missing-config case visible in run history instead of silent. |
| 2-yes | `Get_Created_By_User` | Dataverse: Get a row by ID | Table `systemusers`, Row ID `@triggerOutputs()?['body/_createdby_value']` |
| 2-yes | `Get_Leave_Type` | Dataverse: Get a row by ID | Table `contoso_leavetypes`, Row ID `@triggerOutputs()?['body/_contoso_leavetype_value']` |
| 2-yes | `Compose_Stage_Label` | Compose | `@if(equals(triggerOutputs()?['body/contoso_approvalstatus'], 'Pending HR Approval'), 'approval', 'capture')` |
| 2-yes | `Send_email_to_HR` | Outlook: Send an email (V2) | To `@first(outputs('Get_Settings')?['body/value'])?['contoso_hremail']`. Subject `Leave request pending HR @{outputs('Compose_Stage_Label')}: @{outputs('Get_Created_By_User')?['body/fullname']}`. Body: requester, leave type (`outputs('Get_Leave_Type')?['body/contoso_name']`), start date, end date, duration, comments, last actor `@{triggerOutputs()?['body/contoso_lastactor']}` and their comments (`contoso_lastactorcomments`), plus the app link `@{first(outputs('Get_Settings')?['body/value'])?['contoso_appurl']}` |

### Design notes and weak points

- Fires once per status change. It is independent of `LeaveRequestApproval`, so both run in parallel; neither depends on the other.
- Only one HR address. If approvers and capturers are different teams, add `contoso_hrcaptureemail` and pick the address with the same `if()` used in `Compose_Stage_Label`.
- No error handling of its own. Apply the Try/Catch pattern from Flow 3 if you want failures reported.

---

## Flow 3: Handle Approval Flow Failures

**Purpose:** when anything in `LeaveRequestApproval` fails, email an administrator the run link and the error. This is a change to the existing flow (not a separate one): do it on a copy first ("Save as"), then swap.

### New Dataverse columns

None. The admin address should be an **environment variable** rather than a settings-table column, because the Catch branch must not depend on Dataverse being reachable: create a solution-aware environment variable `contoso_AdminEmail` (Data type Text) and reference it in the flow.

### Trigger

Unchanged: `Approval_Status_Column_Changes` (Dataverse, modified/added on `contoso_leaverequest`, filter `contoso_approvalstatus`).

### Restructured actions in order

Variables cannot be initialised inside a Scope, so the two `Initialize` actions stay at the top level and become empty; their values are set inside the Try.

| # | Action name | Connector | Change from today |
|---|---|---|---|
| 1 | `Initialize_CreatedBy_Email_Address` | Variables | Keep at top level, but **remove the value** (String, empty). Today it reads `outputs('Get_Created_By_User')`, which would now be inside the scope. |
| 2 | `Initialize_Power_App_URL_Variable` | Variables | Unchanged, top level, runs after #1 |
| 3 | `Scope_Try` | Control: Scope | New. Contains everything below, in this order, with the existing `runAfter` chain preserved. |
| 3.1 | `Get_Created_By_User` | Dataverse | Moved into scope, unchanged |
| 3.2 | `Get_Leave_Type` | Dataverse | Moved, unchanged |
| 3.3 | `Set_CreatedBy_Email_Address` | Variables: Set variable | **New.** Name `EmployeeEmailAddress`, value `@outputs('Get_Created_By_User')?['body/internalemailaddress']` |
| 3.4 | `Get_Leave_Manager_Settings_Row` | Dataverse | Moved, unchanged |
| 3.5 | `Apply_to_each` (with `Set_Power_App_URL_Variable`) | Control | Moved, unchanged |
| 3.6 | `Switch` (all six email cases) | Control | Moved, unchanged |
| 4 | `Scope_Catch` | Control: Scope | **New.** Configure run after: `Scope_Try` **has failed** and **has timed out** (leave "is successful" and "is skipped" unticked). |

```json
"Scope_Catch": { "type": "Scope", "runAfter": { "Scope_Try": [ "Failed", "TimedOut" ] }, "actions": { } }
```

Actions inside `Scope_Catch`:

| # | Action name | Connector | Expression / setting |
|---|---|---|---|
| 4.1 | `Filter_array_Failed_Actions` | Data operation: Filter array | From `@result('Scope_Try')`, condition `@or(equals(item()?['status'], 'Failed'), equals(item()?['status'], 'TimedOut'))` |
| 4.2 | `Compose_Failed_Action_Name` | Compose | `@coalesce(first(body('Filter_array_Failed_Actions'))?['name'], 'unknown')` |
| 4.3 | `Compose_Error_Message` | Compose | `@coalesce(first(body('Filter_array_Failed_Actions'))?['error']?['message'], 'No error message returned')` |
| 4.4 | `Compose_Run_URL` | Compose | `@concat('https://make.powerautomate.com/environments/', workflow()?['tags']?['environmentName'], '/flows/', workflow()?['name'], '/runs/', workflow()?['run']?['name'])` |
| 4.5 | `Send_failure_email_to_admin` | Outlook: Send an email (V2) | To the `contoso_AdminEmail` environment variable. Subject `Leave Request Approval flow failed`. Body: failed action `@{outputs('Compose_Failed_Action_Name')}`, error `@{outputs('Compose_Error_Message')}`, run link `@{outputs('Compose_Run_URL')}`, request id `@{triggerOutputs()?['body/contoso_leaverequestid']}`, status `@{triggerOutputs()?['body/contoso_approvalstatus']}`, time `@{utcNow()}` |
| 4.6 | `Terminate_Failed` | Control: Terminate | Status **Failed**, code `APPROVAL_FLOW_FAILED`, message `@{outputs('Compose_Error_Message')}`. Without this the run is reported as **Succeeded**, because the Catch scope itself succeeded. |

### Design notes and weak points

- **"Parallel branch" clarified:** the Catch is a sibling scope that only runs on failure, not a true parallel branch of the Try. Add a parallel branch *inside* `Scope_Catch` if you want a second channel (for example a Teams "Post message in a chat or channel" beside the email).
- **Single point of failure:** the failure email goes out through the same Outlook connector that probably failed (expired connection). A second channel such as Teams covers that.
- **Nested errors:** `result('Scope_Try')` reports only the scope's direct children, so a failed email inside the `Switch` will be reported as the `Switch` failing. I have not verified whether `result('Switch')` returns the inner action; try it in the designer and, if it does, add a second Filter array over it.
- **Environment variable in expressions:** in the designer, pick it from Dynamic content (Parameters). Do not type its name by hand.
- **Optional hardening:** set an explicit retry policy on the Outlook sends and the Dataverse `Get a row` actions (Settings, Retry policy, Exponential, count 4). Nothing sets one today.
- **Same pattern elsewhere:** `CreateHistoryRecord` and `CreateRemoveMeetingRequest` have the same gap and could reuse this Catch.
