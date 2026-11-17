# TC-HR-HIRE-02 — Confirm the pre-hire in Workday with Integration Service

**Test Manager case:** `WDAY:309`  
**Guide step:** WD-01 — Confirm the pre-hire in Workday  
**Workflow:** `Tests/TC-HR-HIRE-02.xaml`  
**Status:** Implemented with the generic UiPath Integration Service Workday connector; validation is clean. Build and runtime execution are intentionally pending.

## 1. Test case header

| Field | Specification |
|---|---|
| ID | `TC-HR-HIRE-02` |
| Objective | Verify that the ATS-created Workday pre-hire matched by the run record's email exists exactly once and that its ID, legal name, email, and phone match the `WorkdayHireRun` record. |
| Application | UiPath Integration Service, SOAP-based `Workday` connector; no browser UI. |
| Connector | Generic `UiPath.IntegrationService.Activities` `ConnectorActivity` using `uipath-workday-workday`; not Workday REST and not a vendor-specific Workday activity package. |
| Workday tenant | `uipath_dpt1` implementation tenant. Authentication is stored in the Integration Service connection; the workflow does not read a credential asset, build SOAP, or handle a password. |
| Lookup operation | **Get Pre-Hire by Email**, using `currentRun.Email` in the connector's `Email_Address` input. The live operation is case-insensitive and returns a `GetPreHireByEmail[]` collection. |
| Workday writes | None. The connector call is read-only. Data Fabric receives only the test-result persistence updates. |

## 2. Integration Service connection discovery

The SOAP Workday connector was inspected across accessible folders. Exactly one enabled Workday connection was visible:

| Connection name | Folder | Connector | State | Default | Account evidence |
|---|---|---|---|---|---|
| Workday integration connection | `Workday` | `Workday` SOAP (`uipath-workday-workday`) | Enabled | Yes | Owned by the project owner; the CLI exposes the connection name and owner but not a separate vendor username. |

The connection ping returned **Enabled / Connection is active**. The connection was not created or modified.

The workflow preserves the Studio-generated, opaque connector `Configuration` and binds the existing connection ID `0194b511-65cb-429b-a0a6-71660b9cbaa7`. The configuration blob is not hand-authored or decoded.

## 3. Arguments and outputs

| Name | Direction | Type | Behavior |
|---|---|---|---|
| `in_RunId` | In | `String` | If supplied, selects that `WorkdayHireRun`; if blank, the newest run is queried. |
| `out_RunId` | Out | `String` | Resolved `WorkdayHireRun.RunId`. |
| `Passed` | Out | `Boolean` | True only when the run lookup, exact-one Workday lookup, and all six WD-01 assertions pass. |
| `SummaryMessage` | Out | `String` | Logs only `RunId`, Pre-Hire ID, stage, and failed assertion titles; never logs email, phone, credentials, or password. |

## 4. Data Fabric fields read and written

Entity: tenant-scoped `WorkdayHireRun`, ID `85e73958-ebde-f111-91f2-002248a179ce`.

| Field | Use |
|---|---|
| `RunId` | Lookup key and output. |
| `PreHireId` | Expected Workday `Applicant_ID`; included in the final summary only after a successful lookup. |
| `FirstName`, `LastName` | Expected legal first and last names. |
| `Email` | Input to `Get Pre-Hire by Email` and expected email value. |
| `Phone` | Expected phone, normalized by removing non-digits and retaining the final ten digits when a country code is present. |
| `Stage` | `PreHireVerified` only when all assertions pass; `Failed` when an assertion or guarded operation fails. |
| `LastTestCase` | `TC-HR-HIRE-02` on both success and failure persistence. |
| `PreHireVerifiedAt` | `DateTimeOffset.Now` on success. |
| `ErrorMessage` | Semicolon-separated failed Verify titles on assertion failure; exception text when a guarded operation raises before titles are available. |

## 5. Workday output field mapping from the live connector

Studio's generated JIT schema for `Get Pre-Hire by Email` exposes the required nested response fields. The XAML binds the operation output to `workdayPreHires` as `GetPreHireByEmail[]`.

| Test value | Generated output path | Implementation |
|---|---|---|
| Result count | `workdayPreHires.Count` / array length semantics | Verify exactly one result before any field comparison. |
| Pre-Hire ID | `workdayPreHires(0).Applicant_ID` | Compared to `currentRun.PreHireId`. |
| First name | `workdayPreHires(0).Personal_Data.Name_Data.Legal_Name_Data.Name_Detail_Data.First_Name` | Compared to `currentRun.FirstName`. |
| Last name | `workdayPreHires(0).Personal_Data.Name_Data.Legal_Name_Data.Name_Detail_Data.Last_Name` | Compared to `currentRun.LastName`. |
| Email | `workdayPreHires(0).Personal_Data.Contact_Data.Email_Address_Data(0).Email_Address` | Compared case-insensitively to `currentRun.Email` using `RegexMatch` with an escaped `(?i)^...$` pattern. |
| Phone | `workdayPreHires(0).Personal_Data.Contact_Data.Phone_Data(0).Phone_Number` | Non-digits are removed; if more than ten digits remain, the final ten digits are compared to the similarly normalized run-record phone. |

The connector output contains nested email and phone collections. The implementation uses the first returned email and phone entry after the exact-one pre-hire gate. A missing phone or email entry evaluates as an empty value and causes its corresponding assertion to fail without logging the contact value.

## 6. Object Repository elements

None. This is a headless Integration Service/Data Fabric test case. No browser, `NApplicationCard`, selector, or Object Repository element is used.

## 7. Targeting and risky points

- Use only the existing Workday integration connection in the `Workday` folder.
- Use the generic `ConnectorActivity` and the SOAP Workday connector; do not use Workday REST, raw HTTP, SOAP XML, `GetRobotCredential`, `SecureString`, or a vendor-specific Workday package.
- Bind `Email_Address` to `[currentRun.Email]`; the generated `GetPreHireByEmail[]` output is bound to `workdayPreHires`.
- Preserve the generated connector `Configuration`, field names, field types, and field list exactly.
- The exact-one Workday Verify is a gate. When it fails, the field Verifies are skipped because there is no safe result to compare.
- Every Verify uses `ContinueOnFailure=True`, screenshots on both success and failure, and the required WD-01 title/message format.
- Assertion failure writes `Stage = Failed`, `LastTestCase = TC-HR-HIRE-02`, and the failed Verify titles to `ErrorMessage`; the workflow sets `Passed=False`.
- Only the all-pass branch writes `Stage = PreHireVerified`, `PreHireVerifiedAt = DateTimeOffset.Now`, and `LastTestCase = TC-HR-HIRE-02`.
- The outer Try/Catch persists guarded-operation exceptions as failed runs and rethrows them.
- Workday is read-only; no create, insert, update, or delete operation is present.

## 8. Given / When / Then

### G1 — Resolve the run

- **Given** the imported `WorkdayHireRun` entity is available.
- **When** resolve `in_RunId`; query the specified run when supplied, otherwise query the newest run.
- **Then** Verify exactly one current run and expose its `RunId` through `out_RunId`.

### G2 — Read the Workday pre-hire by email

- **Given** the selected run contains `Email`, `PreHireId`, `FirstName`, `LastName`, and `Phone`.
- **When** execute the generic Workday `Get Pre-Hire by Email` connector operation using `Email_Address = currentRun.Email`.
- **Then** Verify exactly one pre-hire, its ID, legal first name, legal last name, case-insensitive email, and normalized phone. Skip the field Verifies when the result count is not exactly one.

### G3 — Persist verification outcome

- **Given** the lookup and field Verifies have completed.
- **When** aggregate the Verify results.
- **Then** persist `Failed` with failed titles when any Verify is false; otherwise persist `PreHireVerified`, `PreHireVerifiedAt = now`, and `LastTestCase = TC-HR-HIRE-02`.

## 9. Assertion inventory

All assertions use `UiPath.Testing.Activities` and have `ContinueOnFailure=True`, `TakeScreenshotInCaseOfFailingAssertion=True`, and `TakeScreenshotInCaseOfSucceedingAssertion=True`.

| ID | Alternative verification title | Activity | Assertion |
|---|---|---|---|
| V-01 | `WD-01 step 1 - Workday returns exactly one pre-hire for the lookup` | Verify Expression | `workdayPreHires IsNot Nothing AndAlso workdayPreHires.Length = 1`. |
| V-02 | `WD-01 step 1 - its Pre-Hire ID equals the run record's PreHireId` | Verify Expression With Operator / `Equality` | `Applicant_ID` equals `currentRun.PreHireId`. |
| V-03 | `WD-01 step 2 - first name equals the record's FirstName` | Verify Expression With Operator / `Equality` | Generated legal first name equals `currentRun.FirstName`. |
| V-04 | `WD-01 step 2 - last name equals the record's LastName` | Verify Expression With Operator / `Equality` | Generated legal last name equals `currentRun.LastName`. |
| V-05 | `WD-01 step 3 - email equals the record's Email` | Verify Expression With Operator / `RegexMatch` | First returned email equals `currentRun.Email`, case-insensitive and anchored to the full value. |
| V-06 | `WD-01 step 4 - phone equals the record's Phone` | Verify Expression With Operator / `Equality` | Normalized Workday phone equals normalized `currentRun.Phone`. |

For comparisons, `FirstExpression` is the Workday actual value and `SecondExpression` is the run-record expected value or pattern. The output format is:

`<what is checked>: actual '{LeftExpression}', expected {Operator} '{RightExpression}'. Result: {Result}.`

For V-01, the output format reports the result count without exposing contact data:

`Workday returns exactly one pre-hire for the lookup ('<count>'). Result: {Result}.`

## 10. Assumptions

1. The user-selected lookup is the generic curated **Get Pre-Hire by Email** operation, not the generic ID-based `Get Record` operation.
2. The exact email lookup is treated as case-insensitive, as confirmed by live Workday connector metadata.
3. The first returned email and phone entries are the applicable contact values for this pre-hire after the exact-one result gate.
4. `Phone_Number` is the source field. The workflow removes non-digits and compares the final ten digits when a country code is present, so `+1 312 555 0142` becomes `3125550142`.
5. The generated JIT response type is retained in the Studio-authored XAML. The headless validator cannot independently load that Studio-session-generated output assembly, but the active Studio validation of this file is clean.
6. The obsolete Test Manager credential parameter is not used by this workflow. No project manifest or Test Manager metadata was changed because the requested file scope is limited to `TC-HR-HIRE-02.xaml` and this specification.

## 11. Validation and execution status

- Per-file Studio-backed validation: clean — `No diagnostics found.`
- Project build: intentionally not run because the user explicitly requested no build yet.
- Runtime execution: intentionally not run.
- The next authorized gate is a project build, followed by a runtime execution with the RunId from the latest successful ATS-01 run.
