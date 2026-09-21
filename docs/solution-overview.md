# LeaveManager solution overview (v1.0.0.16)

Generated from the unpacked solution in `unpacked/` (Entities, Roles, Workflows) plus the canvas app sources in `apps_src/`. Nothing here was run against a live environment. Where something is inferred rather than read from a file, it says so.

## 1. Dataverse tables

All seven custom tables are **user/team-owned** (`OwnershipTypeMask` = UserOwned). Two standard tables (`SystemUser`, `aaduser`) are included only so relationships can point at them; they carry no custom columns.

| Table (logical name) | Purpose | Key columns |
|---|---|---|
| **Leave Request** (`contoso_leaverequest`) | One row per leave request. The centre of the solution: every flow triggers off it. | `contoso_employee` (lookup to aaduser), `contoso_leavetype` (lookup), `contoso_startdate`, `contoso_enddate`, `contoso_duration` (decimal), `contoso_singledayduration` (choice), `contoso_leavedetail`, `contoso_persalnumber`, `contoso_supportingdocument` (file). **Workflow state:** `contoso_approvalstatus` (free text), `contoso_approveremail`, `contoso_approverid`. **Audit of last action:** `contoso_lastactor`, `contoso_lastactoraction`, `contoso_lastactorcomments`, `contoso_lastactortimestamp`. **Calendar link:** `contoso_meetingrequestid`. |
| **Leave Request History** (`contoso_leaverequesthistory`) | Append-only log of every action taken on a request (submitted, approved, declined, recalled, captured). | `contoso_leaverequest` (lookup), `contoso_action`, `contoso_actor`, `contoso_actorcomments`, `contoso_name` |
| **Leave Type** (`contoso_leavetype`) | Catalogue of leave types and the rules for each. | `contoso_name`, `contoso_hasbalance`, `contoso_specialtype`, `contoso_allowhourlyrequest`, `contoso_allowstartdateinpast`, `contoso_employeeselectable`, `contoso_default`, `contoso_requiresupportingdocument`, `contoso_requiresupportingdocdays`, `contoso_requiresupportdocinpastdays`, `contoso_leavedescription` |
| **Leave Balance** (`contoso_leavebalance`) | One row per employee, leave type and year. Shown to employees in the app. | `contoso_employee` (lookup), `contoso_leavetype` (lookup), `contoso_year` (text), `contoso_startingallowance`, `contoso_leavetaken`, `contoso_currentbalance`, `contoso_balancepreviousyear` (all plain decimals, not calculated), `contoso_visibletoemployee`, `contoso_suggestedrecordowner` (lookup to systemuser), `contoso_employeeaadid`, `contoso_persalnumber` |
| **Holiday** (`contoso_holiday`) | Public holidays / closures used by the app's working-day calculation. | `contoso_name`, `contoso_holidaydate` (date only), `contoso_holidaytype` (National Public Holiday `463270000` / Organisation Closure `463270001` / Other `463270002`), `contoso_workingday` (Yes/No, default No) |
| **Leave Manager Setting** (`contoso_leavemanagersetting`) | Single configuration row, looked up by `contoso_name eq 'default'`. | `contoso_appurl`, `contoso_leavepolicyurl`, `contoso_maxconsecutivedays`, `contoso_sendmeetingrequest`, `contoso_enableappfeedback`. **There is no HR email or admin email column.** |
| **Employee Info** (`contoso_employeeinfo`) | Maps a Microsoft Entra user to a Dataverse user and a Persal (HR payroll) number. *Purpose inferred from columns; no flow or app in this solution reads it.* | `contoso_employee` (aaduser), `contoso_dataverseuser` (systemuser), `contoso_persalnumber` |
| `aaduser`, `SystemUser` | Standard tables, referenced only. | none custom |

Also in the solution (not tables): five canvas apps (**Request Leave**, the main app; **HR Approve**; **HR Capture**; a Manage Leave default command library; a pop-up dialog app), the **Manage Leave** model-driven app, and four **business rules** (`Set default past days to 0`, `Show Hide Single Day Duration`, `Show hide Past Days field`, `Show hide doc days required`) which only control form fields.

## 2. Relationships

Derived from the lookup columns and the relationship names in `Other/Relationships.xml`. Arrow = "many-to-one, points at".

```
                       +----------------+
   +------------------>|   aaduser      |<---------------------+
   |                   | (Entra user)   |                      |
   |                   +----------------+                      |
   |  contoso_employee        ^                                |
   |                          | contoso_employee               | contoso_employee
+--+---------------+   +------+-----------+          +---------+-------+
| Leave Request    |   | Employee Info    |          | Leave Balance   |
+--+------------+--+   +------+-----------+          +---+-----------+-+
   ^            |             | contoso_dataverseuser    |           |
   |            |             v                          |           | contoso_suggestedrecordowner
   |            |      +--------------+                  |           v
   |            |      |  systemuser  |<-----------------+   (used by the owner-assignment flow)
   |            |      +--------------+
   |            | contoso_leavetype                       | contoso_leavetype
   |            v                                         v
   |      +------------------------------------------------------+
   |      |                     Leave Type                        |
   |      +------------------------------------------------------+
   |
   | contoso_leaverequest
+--+--------------------+
| Leave Request History |
+-----------------------+

Holiday              (no relationships; read by the app for day counting)
Leave Manager Setting (no relationships; single 'default' row, read by flows and app)
```

Two links that are **not** foreign keys: a Leave Request is tied to its Leave Balance only by matching employee + leave type + year in app formulas (for example `LeaveDatesScreen.fx.yaml` line 78), and the approver is stored as text (`contoso_approverid`, `contoso_approveremail`), not as a lookup.

## 3. Security roles

Three roles exist in `unpacked/Roles/`: **Leave Users**, **HR Leave Approvers**, **Leave Admin**. Access levels: **Basic** = only records the user owns, **Global** = every record in the organisation. `-` = no privilege granted. Read from the role XML; roles were not tested in an environment.

| Table | Leave Users | HR Leave Approvers | Leave Admin |
|---|---|---|---|
| Leave Request | Create B, Read B, Write B, Delete B (also Append, Assign, Share at B) | Read G, Write G | - |
| Leave Request History | Read B | Read G | - |
| Leave Balance | Read B | Read G | - |
| Leave Type | Read G | Read G | Create G, Read G, Write G (**no Delete**) |
| Holiday | Read G | - | Create G, Read G, Write G, Delete G |
| Leave Manager Setting | Read G | - | - |
| Employee Info | Read G | - | - |
| aaduser | Read G (plus Append / AppendTo) | - | - |

What stands out:

- **Nobody can create or write** Leave Balance, Leave Manager Setting, Employee Info or Leave Request History. Those rows must come from a system administrator, a data import, or a flow running under its owner's connection.
- **HR Leave Approvers cannot create or delete anything**, and have no read on Holiday or Settings. If HR staff use the main app they presumably also hold Leave Users (an assumption; the solution does not assign roles).
- **Leave Admin has no access to Leave Requests or Balances**, only to Holiday and Leave Type.
- **Leave Users have Write (Basic) on their own requests**, and approval status is a plain text column. Nothing at the data layer stops an employee setting their own request to "Approved" through any route other than the app.
- All three roles also carry identical boilerplate: SharePoint data/document privileges and Read on plugin/SDK-message tables.

## 4. Flows

Four modern cloud flows (`Category` 5), all switched **on** (`StateCode` 1). Each connects to Dataverse via `contoso_DVReference`. Two Outlook connection references are in use: `contoso_OutReference` (approval flow) and `new_sharedoffice365_5d239` (meeting flow). Five other workflow files exist but are not cloud flows: four business rules (form logic only) and one classic workflow, `Change Owner`, which is **switched off** (`StateCode` 0).

**Leave Request Approval** (`LeaveRequestApproval-87367883-…json`). Triggered by `Approval_Status_Column_Changes`: a Dataverse "added or modified" trigger on `contoso_leaverequest`, filtered to the `contoso_approvalstatus` column. It is a *notification* flow, not an approval flow: no Approvals connector is used. It looks up the requester (`Get_Created_By_User`), the leave type (`Get_Leave_Type`) and the `default` settings row (`Get_Leave_Manager_Settings_Row`) to build the app link, then a `Switch` on the status sends one Outlook email: to the approver when the status is *Pending Manager Approval* or *Draft* (recalled), and to the employee for *Pending HR Approval*, *Pending HR Capture*, *Approved* and *Declined*. The real approving is done by the canvas apps writing the status column.

**Create History Record** (`CreateHistoryRecord-F60A7D75-…json`). Triggered by `Leave_Request_Action_Taken`: added or modified on `contoso_leaverequest`, filtered to `contoso_lastactortimestamp`. A single action, `Add_Action_to_History_Table`, creates a `contoso_leaverequesthistories` row copying the last action, actor and comments, linked to the request and owned by the request's owner. Because every app action stamps a new timestamp, this produces a full audit trail.

**Create Remove Meeting Request** (`CreateRemoveMeetingRequest-2DF2245A-…json`). Triggered by `When_leave_request_approval_status_changes`: *modified* (not created) on `contoso_leaverequest`, filtered to `contoso_approvalstatus` with the expression status = `Approved` or `Cancelled`. It reads the settings row and does nothing unless `contoso_sendmeetingrequest` is true. On *Approved* it creates an all-day "Approved time off" Outlook calendar event (`Create_Meeting_Request`, shown as out of office; half-day requests are not all-day) and stores the event id on the request (`Save_Meeting_Request_ID`). On *Cancelled* it deletes that event (`Delete_event_(V2)`). The calendar id and the time zone `(UTC+02:00) Harare, Pretoria` are hardcoded.

**Assign Owner to Leave Balance Record** (`AssignOwnertoLeaveBalanceRecord-E61A3B2E-…json`). Triggered by `When_a_Leave_Balance_record_added`: a Dataverse *create* trigger on `contoso_leavebalance`. One action, `Update_Leave_Balance_with_Employee_and_Owner`, sets the Employee lookup from `contoso_employeeaadid` and the row owner from `contoso_suggestedrecordowner`. Presumably this exists so employees, who can only read their own balances (Basic), can see rows that admins bulk-loaded (inference). It does not touch any balance number.

## 5. Known gaps

Each item below was checked against the files; evidence is given so it can be re-verified.

### The five requested gaps

1. **No escalation.** All four cloud flows have Dataverse row triggers; none has a Recurrence (schedule) trigger. No table has a reminder or escalation column, and the manager's manager is not stored anywhere. A request left in *Pending Manager Approval* waits indefinitely; the only time-related data is `contoso_lastactortimestamp`, which the apps do set on every action.
2. **No HR notification in the Approval flow.** The `Pending_HR_Approval` case (`Send_email_after_HR_approval`) and the `Pending_HR_Capture` case (`Send_an_email_(V2)`) email the **employee** only (manager on Cc for the first). No HR recipient exists, and `contoso_leavemanagersetting` has no HR email column. HR must find work by opening the HR Approve / HR Capture apps.
3. **No error handling.** In all four flows every `runAfter` is empty or `Succeeded`; there are no Scope actions, no Failed / TimedOut branches and no explicit retry policy (platform default retries apply to connector calls). A failed step ends the run as Failed and nobody is told. The approval flow's first two actions (`Get_Created_By_User`, `Get_Leave_Type`) sit in front of every email, so one failure blocks all of them.
4. **Balance updates are unconfirmed and appear to be external.** No flow writes `contoso_leavetaken` or `contoso_currentbalance`. In `apps_src/` (HR Approve, HR Capture, Request Leave) there are 55 `Patch` / `Collect` / `ClearCollect` / `Update` / `Remove` style calls; every `Patch` targets `'Leave Requests'` or a local collection, and none targets `'Leave Balances'`. The balance columns are plain decimals (not calculated or rollup), no role has Create/Write on Leave Balance, and the solution contains no plugin assemblies. The apps only *read* balances (and compute `Current Balance - _requestedDays` for display in `LeaveDatesScreen.fx.yaml` line 101). Whatever decrements them, likely an HR import from Persal, is outside this solution. **Confirm with the solution owner.**

### Further gaps found while checking

6. **Weekend holidays are deducted twice** (BUG-001 in [bug-fixes.md](bug-fixes.md), with the corrected formula). `LeaveDatesScreen.fx.yaml` lines 251-253: `_requestedDays = _workDaysInRequest - _holidaysInRequest`, where `_holidaysInRequest` counts every non-working holiday in the range, weekend or not. A request covering only Saturday 15 Aug 2026 gives 0 - 1 = **-1** days (the Next button stays disabled for 0 or less, line 509). A separate over-count, BUG-002, affects a 5-day leftover part-week starting on a Wednesday (line 236).
7. **"Cancelled" is never set by any app.** It only appears in filter expressions. So the meeting-deletion branch of `Create Remove Meeting Request` cannot be reached from the apps, and the Approval flow has no `Switch` case or email for it.
8. **Approver address is the sign-in name.** `LeaveApproverScreen.fx.yaml` line 273 stores `_defaultApprover.UserPrincipalName` as `contoso_approveremail`, and the flow uses it as an email To address. If a UPN differs from the mailbox address, the email fails or misroutes. If the employee has no manager, the default approver is blank.
9. **Status is a free-text column.** The apps and flows both hardcode the strings ("Pending HR Capture" is a literal in `hrapprove/Src/MainScreen.fx.yaml` line 117). A typo silently falls into the empty `default` case of the `Switch`.
10. **`Send_an_email_(V2)` (Pending HR Capture) has a blank "Submitted by:"** line: the name expression is missing (approval flow, line 193).
11. **Settings row is assumed to exist.** With no `default` row, the approval email link becomes a bare `&approvals=1`, and the meeting flow silently sends nothing (`SendMeetingRequests` stays unset, so the condition is false). No error is raised in either case.
12. **Hardcoded environment values.** Calendar id and `(UTC+02:00) Harare, Pretoria` in the meeting flow; two different Outlook connections across the two flows.
13. **Approval integrity is enforced only in the apps** (see section 3: employees hold Write on their own requests).
14. **"Draft" exists in Dataverse only after a recall.** The app creates requests directly as *Pending Manager Approval* (`ConfirmationScreen.fx.yaml` line 43) and sets *Draft* only on Recall (`HomeScreen.fx.yaml` line 670). So the "recalled" email fires only on a genuine recall.
