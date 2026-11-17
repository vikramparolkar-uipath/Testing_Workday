# TC-HR-HIRE-03 — WD-02 Start Hire Employee as the hiring manager

**Test Manager reference:** `WDAY:310`  
**Test case ID:** `73ce9883-a75b-4a58-952f-24ed4564b9b6`  
**Design status:** In Progress — live Event Details indication is intentionally deferred.

## Objective

Verify the Workday WD-02 flow for starting a hire as hiring manager Logan McNeil while acting on behalf of James Walker, selecting an existing pre-hire, completing the captured Hire Employee form, submitting the hire, and persisting the result to the `WorkdayHireRun` Data Fabric entity.

The first execution submits a real Workday hire and must be separately approved. No execution was performed during authoring.

## Preconditions and constraints

- The existing Microsoft Edge Workday tab/session is open and signed in as Logan McNeil.
- Do not open a new Workday window or tab.
- Do not sign in or sign out and never type a password.
- Existing Object Repository screens and elements are the sole UI target source; no elements were recaptured, duplicated, or hand-selected.
- Workday UI timeout defaults to `00:00:30`.
- The `WorkdayHireRun` entity is installed and provides the current run and pre-hire data.
- Event Details controls are not yet captured. They remain real UI Automation activities marked `TODO Indicate` in `Workflows/Workday/ReadEventDetails.xaml` and must be indicated during an approved live run.

## Given / When / Then

### Given

1. Resolve the supplied `in_RunId`.
2. Query `WorkdayHireRun` by `RunId` using `Workflows/DataFabric/GetHireRun.xaml`.
3. Verify that the current run exists and expose the resolved run ID.
4. Attach to the existing Workday Header and verify the signed-in label starts with `Profile Logan McNeil`.

### When

1. Search Workday for `Start Proxy` and open the captured Search Results target.
2. Start proxy as `James Walker` using the captured Start Proxy screen and elements.
3. Read the selected proxy user and proxy banner; verify the proxy-start gate.
4. Search Workday for `Hire Employee` and open the captured result.
5. Read the supervisory organization and select `Existing Pre-Hire`.
6. Enter the current run's pre-hire ID and read the selected pre-hire.
7. Verify the supervisory organization is `Accounts Payable Department` and the selected pre-hire matches the current run.
8. Confirm the Hire Employee prompt.
9. Complete the captured Hire Employee form using the existing repository targets:
   - hire date from the current run;
   - reason `Fill Vacancy`;
   - current run position ID;
   - employee type `Regular`;
   - location `Chicago`;
   - job profile `Accounting Specialist`;
   - time type `Full time`;
   - job title and business title `Accounting Specialist - AP`;
   - scheduled weekly hours `40`;
   - hire comment containing the run ID.
10. Read the selected position and verify it starts with the current run's `PositionId`.
11. Submit the Hire Employee form only after the verification gates pass.
12. Invoke `ReadEventDetails.xaml` after submit. Its Event Details UI targets remain deferred until approved live indication.
13. Search for `Stop Proxy` and stop the proxy using the captured Stop Proxy target.

### Then

1. Verify the aggregate WD-02 gates.
2. On a passing gate, update the current `WorkdayHireRun` record with:
   - `Stage = HireSubmitted`;
   - `LastTestCase = TC-HR-HIRE-03`;
   - `HireEventStatus = In Progress`;
   - the captured/pending event reference;
   - an empty error message.
3. Set `Passed = True` and build a summary that identifies the run and the deferred Event Details boundary.
4. If the aggregate gate fails, set `Passed = False` and leave failure handling to the external execution template.

## Reusable workflows

| Workflow | Responsibility |
|---|---|
| `Workflows/Workday/EnsureSignedInAsLogan.xaml` | Attach to the existing Workday Header and verify Logan McNeil. |
| `Workflows/Workday/SearchAndOpen.xaml` | Search Workday and open the captured result; `resultText` resolves the captured dynamic result selector. |
| `Workflows/Workday/StartProxy.xaml` | Start proxy as James Walker and verify the proxy banner. |
| `Workflows/Workday/StopProxy.xaml` | Stop the Workday proxy. |
| `Workflows/Workday/SetPromptField.xaml` | Read supervisory organization, select Existing Pre-Hire, and read the selected pre-hire. |
| `Workflows/Workday/ReadEventDetails.xaml` | Deferred Event Details reads; contains intentional `TODO Indicate` UI activities. |
| `Workflows/DataFabric/GetHireRun.xaml` | Query the current `WorkdayHireRun` record by run ID. |
| `Workflows/DataFabric/UpdateHireRun.xaml` | Persist the WD-02 stage and hire-event fields. |

## Object Repository usage

The implementation uses the existing Workday Object Repository references for:

- Header, Search Results, Start Proxy, Stop Proxy, Hire Employee Prompt, and Hire Employee Form screens;
- signed-in user, global search, result title, proxy controls, supervisory organization, Existing Pre-Hire controls, Hire Employee form fields, and Submit.

`StartProxy.xaml` and the hand-authored test-case form cards use the exact repository-exported target payloads because the CLI screen linker could not resolve those hand-authored cards. No selectors were invented or changed.

## Validation status

- Clean per-file validation: `EnsureSignedInAsLogan.xaml`, `SearchAndOpen.xaml`, `StartProxy.xaml`, `StopProxy.xaml`, `SetPromptField.xaml`, `GetHireRun.xaml`, `UpdateHireRun.xaml`, and `TC-HR-HIRE-03.xaml`.
- `ReadEventDetails.xaml` intentionally reports six unconfigured-target diagnostics until approved live indication.
- Project build is blocked by those six expected `ReadEventDetails.xaml` target diagnostics.
- The pre-existing `TC-HR-HIRE-01.xaml` was not modified.

## Approval-required next step

During a separately approved live run, indicate the Event Details screen and controls in order, replace the remaining TODO targets with the captured repository links, validate again, and rebuild. Do not execute the real-hire test before that approval.
