# TC-HR-HIRE-01 — Mark a candidate hired in TalentTrack ATS and send the pre-hire to Workday

**Design status:** Specification updated with the TalentTrack user guide and the imported Data Fabric schema. The existing `WorkdayHireRun` entity is already imported into the project. The project metadata now registers `Tests/TC-HR-HIRE-01.xaml` as the sole active Test Case, and the VB Test Case shell has been created. UIA targets, reusable helper workflows, and the end-to-end implementation remain to be authored.

## 1. Test case header

| Field | Specification |
|---|---|
| ID | `TC-HR-HIRE-01` |
| Title | Mark a candidate hired in the ATS and send the pre-hire to Workday |
| Objective | Verify that a recruiter can create a new candidate, advance the candidate to an accepted offer, mark the candidate hired, and receive a Workday Pre-Hire ID from the live ATS-to-Workday integration. |
| Application | TalentTrack ATS, a local single-page web app |
| Base URL | `http://localhost:4100` |
| Browser/runtime | UiPath Modern UI Automation with the UiPath browser extension in Microsoft Edge; open a new browser window; robot runs on the same machine as TalentTrack ATS |
| Preconditions | TalentTrack ATS is running; no login is required; the header pill reads exactly `Workday: LIVE (uipath_dpt1)`; the ATS can reach Workday's live `Put_Applicant` service; the Microsoft Edge UiPath browser extension is installed and available; the tenant-level `WorkdayHireRun` entity is already imported into the project. |
| Test data | Job `4001`, Accounting Specialist - AP / REQ-AP-1001; candidate Riley Livesync plus a unique run suffix; phone `+1 312 555 0142`; start date `2026-11-23`; base salary `70000`; expected position ID `P-01086`. |
| Out of scope | Inspecting the pre-hire inside Workday; independently calling Workday; validating the Workday hand-off implementation itself; ATS API-driven candidate setup or updates; cleanup or deletion in Workday; changing or resetting other candidates or demo data; updating or deleting any `WorkdayHireRun` record other than the record created by this run. |

### Resolved design decisions

- Candidate suffix: generate `RunSuffix = DateTime.Now.ToString("ddMMyyHHmmss", CultureInfo.InvariantCulture)` at runtime when the input is blank; compute `RunId = "HIRE-" & RunSuffix`; query `WorkdayHireRun` by that exact `RunId` before creation and fail without creating a candidate if it already exists. Existing records are never updated or deleted.
- Data Fabric hand-off: use the already imported tenant-level entity `WorkdayHireRun` (display name `Workday - Hire Run`, entity ID `85e73958-ebde-f111-91f2-002248a179ce`).
- Data Fabric behavior: after the live-mode Verify passes, compute the runtime `RunId`, query only for that exact `RunId` to reject duplicates, then create exactly one record; update only that record at the job-mapping, candidate, and Workday hand-off milestones; query that record by unique `RunId` for the final Verify. No existing record is updated or deleted.
- Existing scaffold registration: register only `Tests/TC-HR-HIRE-01.xaml` as the active test case; do not register the existing blank `TestCase.xaml` scaffold. Leave the scaffold file itself untouched unless a separate cleanup request is made.
- Data Fabric activities: use `Create Entity Record`, `Update Entity Record`, and `Query Entity Records` from the imported Data Service package; do not use HTTP calls for Data Fabric. The entity has already been imported through Manage Entities.
- `Stage` is the imported `HireRunStage` single-choice field. Set `Created`, `AtsHired`, or `Failed` using the generated choice-set value from the imported entity, never by assigning a typed numeric value. The complete choice set is `Created`, `AtsHired`, `PreHireVerified`, `HireSubmitted`, `HireCompleted`, `CleanedUp`, and `Failed`.
- Data Fabric write scope: create and update only the current run's record, identified by unique `RunId`; never write a password, credential, or secret.
- Data Fabric final condition: after the Workday hand-off Verify activities pass, update `PreHireId`, `AtsSyncStatus`, `LastTestCase`, and `Stage = AtsHired`, then query by `RunId` and Verify `Stage = AtsHired` and `PreHireId` equals the ATS hand-off value.
- Data Fabric failure condition: wrap the test steps in `Try/Catch`; on any failure, update this run's `Stage = Failed`, `LastTestCase = "TC-HR-HIRE-01"`, and `ErrorMessage` with the failure message, then `Rethrow`.
- Candidate setup: keep every ATS setup action in the UI; do not use ATS setup endpoints. A read-only ATS API fallback is allowed only to retrieve `AtsApplicationId` or webhook events when the UI cannot expose them reliably.
- Browser: Microsoft Edge, opened in a new browser window.
- Cleanup: leave the live Workday pre-hire in place; do not revert the ATS side.
- Webhook assertion: require both named events among the two newest relevant events, without requiring a relative order; correlate by `application_id`, then verify `pre_hire_id`.
- Mapping and offer assertions: read container text from stable `data-testid` targets and verify the expected values.
- Mark-as-hired state: verify the HTML `disabled` attribute before and after recording the offer.
- Workday evidence: use UiPath Testing Verify activities with `TakeScreenshotIfFailed=True` and `TakeScreenshotIfSucceded=True` for each G6 hand-off check; no separate screenshot or attach activity is required.
- Browser launch: use the supported Modern UI Automation `Use Application/Browser` pattern configured to open a new Microsoft Edge window and scope subsequent ATS targets to that browser instance.
- Workday timeout: fail after 45 seconds with evidence, including the last-known status, `data-status`, Workday message, IDs, and the hand-off panel in view. Do not retry the live hand-off.
- Job ID conversion: convert `JobId` to an invariant zero-decimal Decimal before the Data Fabric create; a conversion failure is a setup failure and must occur before candidate creation.

### Guide-derived ATS facts

- Workday tenant: `uipath_dpt1`.
- Job: Accounting Specialist - AP / `REQ-AP-1001`; expected position `P-01086`.
- Supervisory organization: Accounts Payable Department.
- Successful hand-off status text: `Synced to Workday`; `workday-status` `data-status` values include `synced`, `dry_run`, and `failed`.
- Relevant webhook events: `candidate_hired`, `workday_prehire_synced`, and `workday_prehire_failed`.
- The synchronized event payload includes `application_id` and `pre_hire_id`.
- The guide documents ATS read endpoints, including `GET /v1/candidates/{id}` and `GET /v1/events`; this test may use them only as read-only fallbacks for identifiers/event correlation, never for candidate setup or mutation.

### Live-integration boundary

The test interacts only with TalentTrack ATS and the imported Data Fabric entity. It does not supply, read, or log a Workday credential and does not call Workday directly. The ATS server owns the integration credential and performs the live `Put_Applicant` call after the final confirmation.

## 2. Test case arguments

All arguments are intended to be Test Case arguments. `RunSuffix` is an optional override; when blank, the Test Case generates it at runtime before creating the candidate or Data Fabric record.

| Argument | Type | Default | Notes |
|---|---|---|---|
| `AtsBaseUrl` | `String` | `http://localhost:4100` | Used to open the ATS in a new browser window. |
| `JobId` | `String` | `4001` | Used for the route and job target. |
| `RunSuffix` | `String` | blank | Optional override. When blank, generate `DateTime.Now.ToString("ddMMyyHHmmss", CultureInfo.InvariantCulture)` at runtime. Before candidate creation, reject the run if `RunId = "HIRE-" & RunSuffix` already exists in `WorkdayHireRun`; do not retry with a different suffix automatically. |
| `FirstName` | `String` | `Riley` | Candidate first name. |
| `LastName` | `String` | `Livesync` plus the generated `RunSuffix` | Derived at runtime as `"Livesync" & RunSuffix`. |
| `Email` | `String` | `riley.livesync` plus the generated `RunSuffix` and `@example.com` | Derived at runtime as `"riley.livesync" & RunSuffix & "@example.com"`. |
| `Phone` | `String` | `+1 312 555 0142` | Candidate phone. |
| `StartDate` | `DateTime` | `New DateTime(2026, 11, 23)` | Converted to the native HTML date value `yyyy-MM-dd`; UI value must represent November 23, 2026 regardless of browser locale. |
| `BaseSalary` | `Decimal` | `70000` | Entered into the offer form; displayed currency is normalized before comparison. |
| `ExpectedPositionId` | `String` | `P-01086` | Must match the job mapping and the Workday hand-off panel. |
| `WorkdaySyncTimeout` | `TimeSpan` | `New TimeSpan(0, 0, 45)` | Maximum wait after final confirmation for `workday-status` to appear and become synchronized. |

## 3. Outputs

| Output | Type | Definition |
|---|---|---|
| `Passed` | `Boolean` | Informational output set `True` only when every Verify activity and the final Data Fabric Verify pass; an unhandled failure still fails the Test Case. |
| `SummaryMessage` | `String` | Human-readable result including the candidate identity, stage, Workday status, position ID, Pre-Hire ID, and `RunId` when available. Do not include credentials. |
| `PreHireId` | `String` | Workday Pre-Hire ID returned by the ATS, for example `A04907`. |
| `AtsCandidateId` | `Decimal` | ATS candidate ID from the candidate route, candidate page, or read-only ATS fallback. |
| `ApplicationId` | `Decimal` | Test Case output alias for the internal `AtsApplicationId` value; sourced from the candidate page/ATS or read-only `GET /v1/candidates/{AtsCandidateId}` fallback. |
| `out_RunId` | `String` | `HIRE-` plus the unique `RunSuffix`; passed to TC-HR-HIRE-02 through the test set. |

## 4. Data Fabric run record

### 4.1 Imported entity contract

The entity is already imported into this project through **Manage Entities**. Do not add an import/setup step and do not use HTTP calls for Data Fabric.

| Property | Value |
|---|---|
| Entity name | `WorkdayHireRun` |
| Display name | `Workday - Hire Run` |
| Scope | Tenant |
| Entity ID | `85e73958-ebde-f111-91f2-002248a179ce` |
| Unique key | `RunId` (`STRING`, required, unique, maximum length 100) |
| Stage field | `Stage` (`CHOICE_SET_SINGLE`, required, choice set `HireRunStage`) |
| Stage values used by TC-HR-HIRE-01 | `Created`, `AtsHired`, `Failed` |
| Stage values reserved for later cases | `PreHireVerified`, `HireSubmitted`, `HireCompleted`, `CleanedUp` |

### 4.2 Fields written by TC-HR-HIRE-01

The test creates one record and updates only that record, identified by `RunId`. It never writes credentials, passwords, request secrets, or Workday integration configuration.

| Field | Type | Value/source in this test |
|---|---|---|
| `RunId` | `STRING` | `"HIRE-" & RunSuffix`; unique run identifier and lookup key. |
| `Stage` | `HireRunStage` single choice | `Created` at record creation; `AtsHired` after successful hand-off verification; `Failed` in the failure handler. Use the imported choice-set value, never a typed numeric value. |
| `LastTestCase` | `STRING` | `TC-HR-HIRE-01` on creation and every update. |
| `ErrorMessage` | `STRING` | Blank on success; failure message on the failure update. |
| `FirstName` | `STRING` | `FirstName` test argument. |
| `LastName` | `STRING` | Generated `LastName` using `RunSuffix`. |
| `Email` | `STRING` | Generated `Email` using `RunSuffix`. |
| `Phone` | `STRING` | `Phone` test argument. |
| `AtsJobId` | `DECIMAL` | `JobId` converted to a zero-decimal number (`4001`). |
| `ReqId` | `STRING` | `REQ-AP-1001`. |
| `JobTitle` | `STRING` | `Accounting Specialist - AP`. |
| `HireDate` | `DATE` | `StartDate` / offer start date. |
| `BaseSalary` | `DECIMAL` | `BaseSalary` offer argument. |
| `Currency` | `STRING` | `USD`. |
| `SalaryFrequency` | `STRING` | `Annual`. |
| `PositionId` | `STRING` | Read from the job Workday mapping panel; expected `P-01086`. |
| `SupervisoryOrg` | `STRING` | Read from the job Workday mapping panel; expected `Accounts Payable Department`. |
| `JobProfile` | `STRING` | Read from the job Workday mapping panel; expected `Accounting Specialist - AP`. |
| `Location` | `STRING` | Read from the job Workday mapping panel; expected `Chicago`. |
| `TimeType` | `STRING` | Read from the job Workday mapping panel; expected `Full time`. |
| `EmployeeType` | `STRING` | Read from the job Workday mapping panel; expected `Regular`. |
| `HireReason` | `STRING` | Read from the job Workday mapping panel; expected `New Hire > Fill Vacancy`. |
| `HiringManager` | `STRING` | Read from the job Workday mapping panel; expected `James Walker`. |
| `AtsCandidateId` | `DECIMAL` | Read after candidate creation from the candidate URL/ATS or the permitted read-only fallback. |
| `AtsApplicationId` | `DECIMAL` | Canonical internal/entity field and source of the `ApplicationId` Test Case output; read after candidate creation from the candidate URL/ATS or the permitted read-only fallback. |
| `AtsSyncStatus` | `STRING` | Workday hand-off `workday-status` `data-status` value, normally `synced`. |
| `PreHireId` | `STRING` | Workday Pre-Hire ID read from the ATS hand-off panel. |

Fields reserved for later test cases—`ReservationRunId`, `PreHireVerifiedAt`, `HireEventRef`, `HireEventStatus`, `EmployeeId`, `CompletedAt`, `CleanupAction`, `AtsReverted`, and `CleanedUpAt`—must not be populated or changed by TC-HR-HIRE-01.

### 4.3 Data Fabric lifecycle and failure handling

1. After the live-mode Verify passes, generate `RunSuffix` when blank, derive `LastName` and `Email`, convert `JobId` to an invariant zero-decimal Decimal, and assign `RunId = "HIRE-" & RunSuffix`. Query `WorkdayHireRun` by that exact `RunId`; if any record is returned, fail before candidate creation and do not update the existing record. If no record is returned, create exactly one `WorkdayHireRun` record with `Stage = Created`, `LastTestCase = "TC-HR-HIRE-01"`, the candidate data, offer data, known job ID/requisition/title data, and the `RunId` output. The mapping fields read from the job page may be empty at creation because they are optional; they are populated in the next update.
2. After the job page mapping has been read and verified, update the same record with `PositionId`, `SupervisoryOrg`, `JobProfile`, `Location`, `TimeType`, `EmployeeType`, `HireReason`, and `HiringManager`.
3. After the candidate is added, update the same record with `AtsCandidateId` and `AtsApplicationId`.
4. After the Workday hand-off panel has been read and every hand-off Verify passes, update the same record with `PreHireId`, `AtsSyncStatus`, `LastTestCase`, and `Stage = AtsHired`.
5. Query the entity by the unique `RunId`, then Verify that exactly the current run record is returned, `Stage = AtsHired`, and `PreHireId` equals the Pre-Hire ID read from the ATS. A Data Fabric persistence failure fails the Test Case.
6. Wrap the scenario steps in `Try/Catch`. If any step fails after record creation, update only the current run record with `Stage = Failed`, `LastTestCase = "TC-HR-HIRE-01"`, and the failure text in `ErrorMessage`, then `Rethrow` so the Test Case fails. If record creation itself fails, preserve the original failure and report that the failure record could not be updated because no run record exists.

### 4.4 Data Fabric activities

- `Create Entity Record` creates the initial record.
- `Update Entity Record` updates the same entity record reference at the mapping, candidate, hand-off, and failure milestones.
- `Query Entity Records` retrieves the record by `RunId` for the final verification only.
- The imported Data Service package/entity definitions are the source of truth for generated entity and choice-set types.

## 5. Object Repository design

### 5.1 Application and screens

- **Application:** `TalentTrack ATS`
- **Screens:** `Jobs`, `Job`, `Candidate`, `Webhook log`
- **Naming convention:** `Screen_Element_Type`; examples include `Candidate_MarkAsHired_Button` and `Job_StartDate_Input`.
- **Capture order:** Capture only the targets used by this test, in the order in which the test first needs them. Do not capture the entire application.
- **Primary targeting:** `data-testid` first. Fall back to `id` or `name` only where no suitable `data-testid` target exists. Never target by screen position or visible text alone.
- **Activity type rule:** Inputs use `Type Into`; buttons and links use `Click`; static labels/headings/panels use `Click` as their capture activity even when never clicked; changing values are read with `Get Text`; `workday-status` also uses `Get Attribute` for `data-status`. Runtime state reads such as the `disabled` attribute are additional reads, not selector strategies.

### 5.2 Element capture table

The table is in test-case order. A dynamic element is one logical Object Repository target whose selector is resolved from the current candidate/event identity; its changing visible text is not used as the selector.

| Order | Screen | Object Repository element | `data-testid` target | Activity type | Use |
|---:|---|---|---|---|---|
| 1 | Jobs | `Jobs_WorkdayMode_Status` | `workday-mode` | `Get Text` | Fail fast unless the exact live pill is `Workday: LIVE (uipath_dpt1)`. This is a changing/status value exception to the static-label rule. |
| 2 | Jobs | `Jobs_JobsTable_Panel` | `jobs-table` | `Click` | Stable table anchor/wait target. |
| 3 | Jobs | `Jobs_JobRow4001_Row` | `job-row-4001` | `Click` | Stable row anchor for job 4001. |
| 4 | Jobs | `Jobs_JobLink4001_Link` | `job-link-4001` | `Click` | Opens the Accounting Specialist - AP job page. |
| 5 | Job | `Job_WorkdayMapping_Panel` | `job-workday-mapping` | `Click` | Static mapping-panel anchor and visibility assertion target. |
| 6 | Job | `Job_PositionId_Text` | `job-position-id` | `Get Text` | Reads the mapped position ID; verify `P-01086`. |
| 7 | Job | `Job_AddCandidate_Form` | `add-candidate-form` | `Click` | Static form anchor/wait target. |
| 8 | Job | `Job_FirstName_Input` | `input-first-name` | `Type Into` | Enters `FirstName`. |
| 9 | Job | `Job_LastName_Input` | `input-last-name` | `Type Into` | Enters `LastName`. |
| 10 | Job | `Job_Email_Input` | `input-email` | `Type Into` | Enters `Email`. |
| 11 | Job | `Job_Phone_Input` | `input-phone` | `Type Into` | Enters `Phone`. |
| 12 | Job | `Job_AddCandidateSubmit_Button` | `add-candidate-submit` | `Click` | Creates the candidate. |
| 13 | Job | `Job_PipelineCandidate_DynamicRow` | `pipeline-candidate-<candidateId>` | `Click` | Dynamic fallback/reselection target after creation; never use the visible candidate name as the selector. |
| 14 | Candidate | `Candidate_CandidateName_Text` | `candidate-name` | `Get Text` | Reads the changing candidate name and supports identity verification/logging. |
| 15 | Candidate | `Candidate_StageStepper_Panel` | `stage-stepper` | `Click` | Static stage-stepper anchor. |
| 16 | Candidate | `Candidate_ApplicationReview_Step` | `stage-application-review` | `Click` | Static stage label used to confirm the initial stage when needed. |
| 17 | Candidate | `Candidate_PhoneScreen_Step` | `stage-phone-screen` | `Click` | Static stage label/anchor for the first advancement. |
| 18 | Candidate | `Candidate_Interview_Step` | `stage-interview` | `Click` | Static stage label/anchor for the second advancement. |
| 19 | Candidate | `Candidate_Offer_Step` | `stage-offer` | `Click` | Static stage label/anchor for the offer stage. |
| 20 | Candidate | `Candidate_Hired_Step` | `stage-hired` | `Click` | Static stage label/anchor for the hired stage. |
| 21 | Candidate | `Candidate_CurrentStage_Text` | `current-stage` | `Get Text` | Reads the changing current stage for Application Review, Interview, Offer, and Hired assertions. |
| 22 | Candidate | `Candidate_AdvanceStage_Button` | `advance-stage` | `Click` | Advances one stage at a time. Selector is anchored to `data-testid`, never to the changing button text. |
| 23 | Candidate | `Candidate_Offer_Form` | `offer-form` | `Click` | Static offer-form anchor/wait target. |
| 24 | Candidate | `Candidate_StartDate_Input` | `input-start-date` | `Type Into` | Native HTML date input; use a locale-independent DOM-compatible date entry method described in §5. |
| 25 | Candidate | `Candidate_Salary_Input` | `input-salary` | `Type Into` | Enters `BaseSalary`. |
| 26 | Candidate | `Candidate_SaveOffer_Button` | `save-offer` | `Click` | Records the accepted offer; later text may be `Update offer`. |
| 27 | Candidate | `Candidate_OfferDetails_Panel` | `offer-details` | `Click` | Static offer-details anchor and accepted-offer verification target. |
| 28 | Candidate | `Candidate_OfferStartDate_Text` | `offer-start-date` | `Get Text` | Reads the changing displayed start date and normalizes it to ISO date. |
| 29 | Candidate | `Candidate_OfferSalary_Text` | `offer-salary` | `Get Text` | Reads the changing displayed salary and normalizes currency/number formatting. |
| 30 | Candidate | `Candidate_MarkAsHired_Button` | `mark-as-hired` | `Click` | Verify disabled before an offer, then verify enabled after the offer; click only when enabled. |
| 31 | Candidate | `Candidate_ConfirmationConfirm_Button` | `modal-confirm` | `Click` | Confirms the live hire hand-off. |
| 32 | Candidate | `Candidate_ConfirmationCancel_Button` | `modal-cancel` | `Click` | Captured as the paired dialog control for repository completeness, but not clicked by this test. |
| 33 | Candidate | `Candidate_WorkdayPanel` | `workday-panel` | `Click` | Static hand-off panel anchor and screenshot target. |
| 34 | Candidate | `Candidate_WorkdayStatus_Text` | `workday-status` | `Get Text; Get Attribute` | Reads changing status text and `data-status`; attribute must equal `synced`. |
| 35 | Candidate | `Candidate_WorkdayPreHireId_Text` | `workday-prehire-id` | `Get Text` | Reads the changing Pre-Hire ID. |
| 36 | Candidate | `Candidate_WorkdayPositionId_Text` | `workday-position-id` | `Get Text` | Reads the changing hand-off position ID. |
| 37 | Candidate | `Candidate_WorkdayHireDate_Text` | `workday-hire-date` | `Get Text` | Reads the changing hand-off hire date. |
| 38 | Candidate | `Candidate_WorkdayMessage_Label` | `workday-message` | `Click` | Static/dynamic panel message anchor; included for failure diagnostics, not used as the sync verdict. |
| 39 | Webhook log | `WebhookLog_NavEvents_Link` | `nav-events` | `Click` | Opens the webhook log. |
| 40 | Webhook log | `WebhookLog_EventsTable_Panel` | `events-table` | `Click` | Stable table anchor/wait target. |
| 41 | Webhook log | `WebhookLog_Event_DynamicRow` | `event-<id>` | `Get Text` | Reads event name and payload from each dynamic event row; the event row's `data-testid` is the anchor, not its visible text. |

`nav-jobs` and `nav-candidates` are not captured for this test because the initial Jobs page is opened directly by URL and the candidate navigation link is not used. `job-title` is not a separate interaction target because the stable `job-link-4001` target opens the required job; if the title must be independently verified, capture it as `Jobs_JobTitle_Label` with activity type `Click`.

## 6. Targeting strategy and risky points

1. **Stable selectors first.** Every target uses its `data-testid` as the primary selector. `id` or `name` is a fallback only when the target has no usable `data-testid`. Visible text is used for verification, not targeting.
2. **Live-mode gate.** Wait for `workday-mode`, read its text, and use a Verify activity against `Workday: LIVE (uipath_dpt1)`. A dry-run or other value fails with a clear setup message before any candidate is created.
3. **Changing advance button.** `advance-stage` remains the target across clicks. The implementation must not select `Move to Phone Screen`, `Move to Interview`, or `Move to Offer` by text. After each click, wait for the current-stage value to change and verify the expected stage.
4. **Native date input.** The value must be entered as the HTML date value `2026-11-23`, not as localized display text such as `11/23/2026`. Use the UI Automation activity's date-input-compatible entry path against `input-start-date`; if browser behavior requires keyboard entry, type the ISO value and verify the rendered offer date after save. Do not depend on the machine/browser locale.
5. **Disabled Mark as Hired.** Do not click the control before the offer. Verify its enabled/disabled state through the target's control state. After saving the offer, wait for the toast to clear and verify that the same `mark-as-hired` target is enabled before clicking.
6. **Confirmation dialog.** Click only `modal-confirm` after the dialog is present. `modal-cancel` is not invoked. The confirmation is the point at which the ATS makes the live Workday call.
7. **Workday wait and spinner.** After `modal-confirm`, wait for the hire action to leave its sending state and for `workday-status` to appear, using element waits/Check App State with a maximum of `WorkdaySyncTimeout` (default 45 seconds). Do not use a fixed Delay. The normal spinner text is `Sending to Workday…`; it is a wait-state diagnostic, not the success assertion.
8. **Toast.** After candidate creation and offer save, wait for the toast to clear before the next click. Do not use a fixed Delay or treat toast disappearance as the business result.
9. **Dynamic pipeline IDs.** `pipeline-candidate-<id>` is resolved only after the ATS creates the candidate. The test must use the generated `AtsCandidateId` or a stable `data-testid` prefix/attribute strategy, never the candidate's visible name or row position.
10. **Dynamic webhook IDs and fallback.** Use the Webhook log UI first. Correlate the relevant events by `application_id`, then verify the synchronized event's `pre_hire_id` against `PreHireId`. If the UI cannot enumerate or expose the dynamic `event-<id>` rows reliably, use the read-only ATS `GET /v1/events` fallback; never use an ATS mutation endpoint for this fallback. Do not hard-code an event ID or select by visible event name alone.
11. **Screenshot evidence.** Before the first G6 hand-off Verify, wait for `workday-status` and the toast to clear, then scroll `workday-panel` fully into view. Each Step 6 UiPath Testing Verify activity sets `TakeScreenshotIfFailed=True` and `TakeScreenshotIfSucceded=True`; those Verify screenshots are the test-result evidence. No separate screenshot or attachment activity is used, and the screenshot must not expose a credential.
12. **Logging.** Log `RunId`, `AtsCandidateId`, `AtsApplicationId`, and `PreHireId`. Never log or expose Workday credentials or request secrets. `workday-request-xml` is not read by this test unless needed for failure diagnostics; it is out of the pass condition.
13. **Data Fabric failure path.** Use a `Try/Catch` around the scenario. On any failure after record creation, update only the current `WorkdayHireRun` record to `Stage = Failed` and store the failure text in `ErrorMessage`, then rethrow. If creation itself fails, preserve the original exception and report that no failure update was possible.

## 7. Given / When / Then steps

### G0 — Live integration precondition and Data Fabric record creation

- **Given** TalentTrack ATS is running at `AtsBaseUrl`, the robot is on the same machine, and `WorkdayHireRun` is imported.
- **When** Open the ATS in a new browser window and wait for `workday-mode`.
- **Then** Verify the header text equals `Workday: LIVE (uipath_dpt1)`. If it does not, set `Passed = False`, produce a clear setup-failure summary, and stop without creating a candidate or Data Fabric record.
- **When** the live-mode Verify passes, generate `RunSuffix` if blank with `DateTime.Now.ToString("ddMMyyHHmmss", CultureInfo.InvariantCulture)`, derive `LastName = "Livesync" & RunSuffix` and `Email = "riley.livesync" & RunSuffix & "@example.com"`, convert `JobId` invariantly to a zero-decimal Decimal, and assign `RunId = "HIRE-" & RunSuffix`.
- **Then** use `Query Entity Records` filtered by the exact `RunId`; Verify that no record exists. If a duplicate exists, fail before candidate creation without changing the existing record.
- **When** the duplicate guard passes, use `Create Entity Record` to create exactly one `WorkdayHireRun` record with `Stage = Created`, `LastTestCase = "TC-HR-HIRE-01"`, the candidate data, offer data, known job data, and `RunId`.
- **Then** set `out_RunId = RunId` and retain the created entity-record reference for all later updates. Do not create a second record.

### G1 — Open the Accounting Specialist - AP job and persist mapping data

- **Given** the live-mode Verify and initial `WorkdayHireRun` creation passed.
- **When** Click `Jobs_JobLink4001_Link` (`job-link-4001`).
- **Then** Verify the `job-workday-mapping` panel is visible.
- **When** Read the mapping container text and extract PositionId, SupervisoryOrg, JobProfile, Location, TimeType, EmployeeType, HireReason, and HiringManager.
- **Then** run eight separate Verifies: PositionId = `P-01086`; SupervisoryOrg = `Accounts Payable Department`; JobProfile = `Accounting Specialist - AP`; Location = `Chicago`; TimeType = `Full time`; EmployeeType = `Regular`; HireReason = `New Hire > Fill Vacancy`; HiringManager = `James Walker`.
- **When** the eight mapping Verifies pass, use `Update Entity Record` on the current run and persist all eight mapping fields.

### G2 — Create the candidate and persist ATS identifiers

- **Given** the job mapping Verify and mapping-field update passed.
- **When** Enter `FirstName`, `LastName`, `Email`, and `Phone` in the Add candidate form, then click `Job_AddCandidateSubmit_Button`.
- **Then** Wait for the candidate profile, read `Candidate_CandidateName_Text`, obtain `AtsCandidateId` from the candidate route/page, and obtain `AtsApplicationId` from the UI or the permitted read-only `GET /v1/candidates/{AtsCandidateId}` fallback. Verify the candidate is at `Application Review` and `Candidate_MarkAsHired_Button` has the HTML `disabled` attribute.
- **When** use `Update Entity Record` on the current run.
- **Then** persist `AtsCandidateId` and `AtsApplicationId` as zero-decimal Decimal fields.

### G3 — Advance to Interview

- **Given** the newly created candidate is at Application Review and the candidate identifiers are persisted.
- **When** Click `Candidate_AdvanceStage_Button`, wait for the stage to change, click it again, and wait again.
- **Then** Verify `Candidate_CurrentStage_Text` reads `Interview`. The two transitions must be Phone Screen, then Interview; the button is always targeted by `advance-stage`, not its changing text.

### G4 — Record the accepted offer

- **Given** the candidate is at Interview.
- **When** Enter `StartDate` into the native date field using the locale-independent approach and enter `BaseSalary`; click `Candidate_SaveOffer_Button` using only its `data-testid` target `save-offer`.
- **Then** Wait for the toast to clear. Verify the stage is `Offer`, the offer details container text includes `Accepted`, the offer start date normalizes to `2026-11-23`, the offer salary normalizes to `70000`, and `Candidate_MarkAsHired_Button` does not have the HTML `disabled` attribute. The later `Update offer` text is not used for targeting.

### G5 — Confirm the live hire

- **Given** an accepted offer exists and Mark as Hired is enabled.
- **When** Click `Candidate_MarkAsHired_Button`, wait for the confirmation dialog, and click `Candidate_ConfirmationConfirm_Button`.
- **Then** Wait for the sending state to resolve and verify `Candidate_CurrentStage_Text` reads `Hired`.

### G6 — Read and verify the Workday hand-off and update Data Fabric

- **Given** the ATS is waiting for the live Workday response.
- **When** Wait up to `WorkdaySyncTimeout` for `workday-status` to appear, wait for the toast to clear, scroll `workday-panel` fully into view, and read the hand-off values.
- **Then** run separate UiPath Testing Verify activities for status text, `data-status`, Pre-Hire ID format, position ID, and hire date. Each G6 hand-off Verify has `TakeScreenshotIfFailed=True` and `TakeScreenshotIfSucceded=True` so its screenshot is test-result evidence. Use the supported UI Automation bring-into-view/scroll action for `workday-panel`; do not use a fixed Delay.
- **When** every G6 hand-off Verify passes, update the current `WorkdayHireRun` record with `PreHireId`, `AtsSyncStatus = "synced"`, `LastTestCase = "TC-HR-HIRE-01"`, and `Stage = AtsHired` using the imported choice-set value.
- **Then** retain `PreHireId` as an output and continue to the webhook and final Data Fabric checks.
- **When** `workday-status` becomes `failed` or does not become synchronized within 45 seconds, capture the last-known status/message through the failure evidence path, update `Stage = Failed` and `ErrorMessage`, then rethrow; do not retry the live hand-off.

### G7 — Verify webhook events and the persisted run record

- **Given** the synchronized hand-off has been verified and the current run has `AtsApplicationId` and `PreHireId`.
- **When** Click `WebhookLog_NavEvents_Link`, wait for `events-table`, and inspect the newest relevant events through the UI; use read-only `GET /v1/events` only if the UI cannot enumerate the dynamic rows reliably.
- **Then** Verify that the two newest relevant events include both `workday_prehire_synced` and `candidate_hired`, regardless of relative order; correlate them by `application_id`, and verify that the synchronized event's `pre_hire_id` equals `PreHireId`.
- **When** the webhook assertions pass, use `Query Entity Records` filtered by the unique `RunId`.
- **Then** Verify exactly one current `WorkdayHireRun` record is returned, `Stage = AtsHired`, and `PreHireId` equals the ATS hand-off value. Set `Passed = True` and build `SummaryMessage` without credentials.

### Failure path for G0–G7

- Wrap G0–G7 in `Try/Catch` after the initial live-mode gate.
- On any failure after `WorkdayHireRun` creation, update only the current record with `Stage = Failed`, `LastTestCase = "TC-HR-HIRE-01"`, and the failure text in `ErrorMessage`, then `Rethrow`.
- If the initial record creation fails, preserve the original failure and report that no failure record could be updated because no current record reference exists.

## 8. Assertion inventory

Every normal-path expected result has its own Verify activity. `V-25` is a failure-path diagnostic performed by the Catch handler or a separate controlled fault test, not by the normal successful run. The following IDs are the planned assertion units; a failure message should identify the target, expected value, and actual value where available.

| Verify ID | Expected result |
|---|---|
| `V-01` | Header `workday-mode` equals `Workday: LIVE (uipath_dpt1)`. |
| `V-02` | `job-workday-mapping` is visible on the job page. |
| `V-03` | Job mapping includes `Accounts Payable Department`. |
| `V-04` | Job position ID equals `P-01086`. |
| `V-05` | Candidate name equals the generated `FirstName + " " + LastName`. |
| `V-06` | Initial current stage equals `Application Review`. |
| `V-07` | Mark as Hired is disabled before an offer is recorded. |
| `V-08` | Current stage equals `Interview` after the two requested advances. |
| `V-09` | Current stage equals `Offer` after the offer is recorded. |
| `V-10` | Offer details show `Accepted`. |
| `V-11` | Offer start date equals `2026-11-23` after normalization. |
| `V-12` | Offer salary equals `70000` after currency/number normalization. |
| `V-13` | Mark as Hired is enabled after an accepted offer. |
| `V-14` | Current stage equals `Hired` after final confirmation. |
| `V-15` | Workday status text equals `Synced to Workday`. |
| `V-16` | `workday-status` attribute `data-status` equals `synced`. |
| `V-17` | Pre-Hire ID matches `^A\d{5,}$`. |
| `V-18` | Workday hand-off position ID equals the job position ID and `ExpectedPositionId` (`P-01086`). |
| `V-19` | Workday hire date equals the offer start date (`2026-11-23`). |
| `V-20` | The two newest webhook events include both `workday_prehire_synced` and `candidate_hired`, without requiring a relative order. |
| `V-21` | The `pre_hire_id` in the `workday_prehire_synced` payload equals `PreHireId`. |
| `V-22` | `Query Entity Records` filtered by the unique `RunId` returns exactly one `WorkdayHireRun` record. |
| `V-23` | The queried `WorkdayHireRun.Stage` choice equals `AtsHired`. |
| `V-24` | The queried `WorkdayHireRun.PreHireId` equals the `PreHireId` read from the ATS hand-off panel. |
| `V-25` *(failure-path diagnostic)* | In the Catch path, the current `WorkdayHireRun` record is written with `Stage = Failed` and a non-empty `ErrorMessage` if any post-creation step fails. This is not a normal success-path Verify. |

## 9. Planned implementation

### 9.1 Entry test case

- `Tests/TC-HR-HIRE-01.xaml` — the sole entry **Test Case** for this scenario, created with the UiPath Test Case template/type rather than as a generic Workflow, using Visual Basic expressions and the test-case arguments/outputs above.
- No `Main.xaml` entry point is used, created, or required; the Test Case itself is the entry point.
- Register only `Tests/TC-HR-HIRE-01.xaml` as an active `TestCase` entry in the project test-case registration and link it to Test Manager test case `WDAY:308`. Do not register the existing blank `TestCase.xaml` scaffold.
- Any future scenario-level test workflow created under `Tests/` must also be created as a **Test Case**; reusable helper components under `Workflows/ATS/` are invoked by the test case and are not independent test cases unless explicitly requested.

### 9.2 Reusable workflows

| Workflow | Responsibility | Main outputs |
|---|---|---|
| `Workflows/ATS/OpenJob.xaml` | Open the ATS, enforce the live-mode precondition, open job 4001, and verify the mapping. | Job context, position ID, pass/fail. |
| `Workflows/ATS/AddCandidate.xaml` | Fill Add candidate, submit, wait for the profile/toast, identify the new candidate/application, and verify initial state. | Candidate ID, application ID, pass/fail. |
| `Workflows/ATS/AdvanceToStage.xaml` | Click `advance-stage` the required number of times, wait for state changes, and verify the target stage. | Current stage, pass/fail. |
| `Workflows/ATS/RecordOffer.xaml` | Enter the locale-independent date and salary, save the offer, wait for the toast to clear, and verify accepted offer/enabled hire action. | Normalized offer values, pass/fail. |
| `Workflows/ATS/MarkAsHired.xaml` | Verify enabled state, open and confirm the modal, wait for the sending state to resolve, and verify Hired. | Stage, pass/fail. |
| `Workflows/ATS/ReadWorkdayHandoff.xaml` | Wait up to the supplied timeout, bring `workday-panel` into view, read status/attribute/Pre-Hire/position/date, and run the G6 hand-off Verify activities with success/failure screenshots enabled. | Pre-Hire ID, status values, screenshot evidence, pass/fail. |
| `Workflows/ATS/ReadWebhookEvents.xaml` | Open the Webhook log, inspect dynamic event rows through the UI, fall back to read-only `GET /v1/events` only when necessary, correlate by `application_id`, and verify the matching `pre_hire_id`. | Event summary, pass/fail. |
| `Workflows/ATS/RecordWorkdayHireRun.xaml` | Create the imported `WorkdayHireRun` record, update the current record at mapping/candidate/hand-off/failure milestones, and query it by `RunId` for final verification using Data Service activities. | Entity record reference, `RunId`, persistence status, pass/fail. |

All reusable workflows will use modern UI Automation activities, element waits, Check App State, and Verify activities. Fixed Delays are excluded.

### 9.3 Build and execution gates after approval

1. Keep the existing Portable Visual Basic Test Automation project configuration; author `Tests/TC-HR-HIRE-01.xaml` with Visual Basic expressions as the sole entry test case. Do not create or require `Main.xaml`.
2. Confirm the installed UI Automation package and Edge browser extension, then capture the Object Repository targets through the supported UiPath target-capture flow; do not hand-author selectors.
3. Author the reusable workflows and the test case in test-case order, then register the test case in `project.json` and link it to Test Manager case `WDAY:308`.
4. Run per-file `uip rpa validate` after each workflow/test-case change and project-level `uip rpa build` after all touched files validate cleanly.
5. After the user-approved build, run the test case with `uipcli`/the project test-run command on the same machine as TalentTrack ATS and report the result summary, assertion results, output IDs, and any attached screenshot evidence. The live run creates a Workday pre-hire; no Workday cleanup is included.

## 10. Capture-time implementation notes

1. **Capture-time verification:** During supported UI target capture, verify the concrete UI source for `AtsCandidateId`/`AtsApplicationId` and the dynamic Webhook log rows. Use the UI first; if the UI cannot expose them reliably, use only the approved read-only fallbacks `GET /v1/candidates/{AtsCandidateId}` and `GET /v1/events`. No candidate setup or mutation API is permitted.
