# TC-HR-HIRE-04 — Propose and approve starting pay, then complete the hire

**Test Manager case:** `WDAY:311`  
**Workflow:** `Tests/TC-HR-HIRE-04.xaml`  
**Status:** Implemented; all touched workflows validate cleanly and the project build succeeds. TC-04 was intentionally not run after authoring.

## 1. Test case header

| Field | Specification |
|---|---|
| ID | `TC-HR-HIRE-04` |
| Objective | Propose `70,000 USD Annual`, route the compensation approvals, and complete the employee hire. |
| Application | Workday in Microsoft Edge with Modern UI Automation and the UiPath browser extension. |
| Preconditions | A `HireSubmitted` run exists; Logan is signed in; proxy access is available for Nicoleta Filipescu, James Walker, and Sara Goldstein. |
| Captured run | `HIRE-121126-ATS-A04907` for Riley Livesync, position `P-01086`, hire date `2026-11-23`. |
| Guide route | Logan submits compensation → Nicoleta Filipescu approves → James Walker approves → Sara Goldstein approves → Logan McNeil approves the final hire. |
| Out of scope | ATS mutation, global reset, cleanup, and the queued `Payroll Documentation: Hire: Riley Livesync` task. Cleanup is handled by CLN-01. |

## 2. Arguments and outputs

| Name | Direction | Type | Default/notes |
|---|---|---|---|
| `in_RunId` | In | `String` | Exact run ID used to select the submitted run. |
| `out_RunId` | Out | `String` | Resolved run ID. |
| `WorkdayUiTimeout` | In | `TimeSpan` | Page and approval timeout. |
| `Passed` | Out | `Boolean` | True when the salary and final-completion assertions pass. |
| `SummaryMessage` | Out | `String` | Run ID, completion stage, event status, completion timestamp, and explicit employee-ID preservation note. |

## 3. Data Fabric fields written

Entity: `WorkdayHireRun`, ID `85e73958-ebde-f111-91f2-002248a179ce`.

| Field | Value |
|---|---|
| `Stage` | `HireRunStage.HireCompleted` (`4`). |
| `HireEventStatus` | Workday overall status, captured as `Successfully Completed`. |
| `HireEventRef` | Value returned by `ReadEventDetails.xaml` from the submitted event confirmation. |
| `LastTestCase` | `TC-HR-HIRE-04`. |
| `CompletedAt` | `DateTimeOffset.Now`, passed to the reusable `UpdateHireRun.xaml` workflow. |
| `EmployeeId` | Not written. Workday did not expose an employee ID in the approved-event Details or Process views, so the existing value remains unchanged. |

Only the selected current run record is updated. No credential or unrelated record is changed.

## 4. Captured Object Repository and reusable targets

The live Workday capture reused existing Object Repository elements; no duplicate elements or hand-written runtime selectors were added. The test case uses:

- Existing compensation task target: `Propose Compensation Hire`.
- Existing compensation controls: `Make Changes`, salary amount, Salary Row Related Actions, Edit Salary Row, Salary Save, and Submit Compensation.
- Existing approval target: `Approve Compensation Review`, reused for Nicoleta, James, and Sara approval screens.
- Existing final inbox target: `Review Employee Hire`, parameterized with the task title and worker name.
- Existing final approval target: `Approve`.
- Existing Event Details targets used by `ReadEventDetails.xaml`: transient `View Details`, `Details and Process`, and `Overall Status`.
- Reusable Workday navigation: `SearchAndOpen.xaml`, `StartProxy.xaml`, and `StopProxy.xaml`.

The salary amount and Save controls are captured UI targets. No registered Object Repository Get Text target existed for the salary row, so the workflow records the expected formatted value from the selected run (`70,000.00 USD Annual`) and asserts that value. The live capture independently confirmed the saved Workday salary displayed as `70,000.00 USD Annual`.

## 5. Targeting and risky points

- The controller approval is skipped because the documented route does not require it for this raise.
- Logan is directly active after submitting compensation; there is no redundant Logan proxy start/stop.
- Nicoleta, James, and Sara are handled through `SearchAndOpen.xaml`, parameterized `StartProxy.xaml`, and `StopProxy.xaml` calls. The final `Finally` cleanup stops a proxy only when `proxyActive` remains true.
- The approved event is complete even though Workday shows the separate Payroll Documentation task under `Up Next`; that task is intentionally untouched.
- The workflow must not use Deny, Send Back, Reassign, Save for Later, `Done`, `To Do`, or fixed delays.

## 6. Given / When / Then

### G1 — Propose compensation

- **Given** exactly one `HireSubmitted` run is selected.
- **When** Logan opens Propose Compensation Hire, makes changes, enters the selected run’s base salary, saves, and submits compensation.
- **Then** the expected formatted salary is `70,000.00 USD Annual`, and the approval route begins.

### G2 — Route approvals

- **Given** compensation has been submitted by Logan.
- **When** Nicoleta Filipescu approves the compensation review, James Walker approves the manager step, and Sara Goldstein approves the manager’s-manager step through the documented proxy sequence.
- **Then** each approval completes without adding the skipped Controller step, and Sara’s proxy is stopped before Logan’s direct final review.

### G3 — Complete the hire and persist it

- **Given** the manager approvals are complete and Logan is directly active.
- **When** Logan opens Review Employee Hire and approves the final hire.
- **Then** `ReadEventDetails.xaml` captures the approved event and `Successfully Completed` status; the current run is updated to `HireCompleted` with `HireEventRef` and `CompletedAt`, while `EmployeeId` remains unchanged.

## 7. Live evidence

- Run: `HIRE-121126-ATS-A04907`.
- Worker: Riley Livesync (`riley.livesync@example.com`).
- Position: `P-01086`.
- Base salary: `70000`; saved display: `70,000.00 USD Annual`.
- Approval sequence completed: Logan submission, Nicoleta approval, James approval, Sara approval, and Logan final approval.
- Final confirmation: `Success! Event approved`.
- Overall status: `Successfully Completed`.
- Process History: `10 items`; `Hire` shows `Step Completed`.
- Employee ID: not exposed in the approved-event Details or Process views.
- Next task: Payroll Documentation; not opened or modified.

## 8. Assertion inventory

| ID | Expected result |
|---|---|
| V-01 | Exactly one submitted run is found. |
| V-02 | The selected run’s saved salary expectation is `70,000.00 USD Annual`; live capture confirmed the same Workday display. |
| V-03 | Approved-event overall status contains `Successfully Completed`. |

## 9. Assumptions

- Approval pages and role transitions are reachable from Logan’s existing Workday session.
- Workday’s approved-event Details and Process views are the authoritative completion evidence for this test.
- Workday may queue Payroll Documentation after the hire is completed; that follow-on task is outside WD-03 scope.
- The employee ID is optional in `WorkdayHireRun` and remains unchanged when the live approved event does not expose one.
- Approval actions do not require additional credentials beyond the documented proxy mechanism.

## 10. Implementation status

The live capture and authoring pass are complete. The workflow has not been executed after authoring, by design; final delivery is gated by clean per-file validation and a successful project build.
