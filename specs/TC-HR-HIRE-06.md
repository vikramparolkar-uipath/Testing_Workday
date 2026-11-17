# TC-HR-HIRE-06 — CLN-01 Clean up after the run

**Test Manager case:** `WDAY:313`  
**Workflow:** `Tests/TC-HR-HIRE-06.xaml`  
**Purpose:** Clean one eligible Workday hire and revert only its corresponding ATS candidate, without using global demo reset or proxy/sign-in actions.

## 1. Contract

| Field | Value |
|---|---|
| ID | `TC-HR-HIRE-06` |
| Test Manager | `WDAY:313` |
| Runtime | Portable, Visual Basic expressions, Modern UI Automation |
| `in_RunId` | Optional exact `WorkdayHireRun.RunId`; blank selects the newest eligible record. |
| `AtsBaseUrl` | Defaults to `http://localhost:4100`. |
| `WorkdayUiTimeout` | Defaults to 30 seconds. |
| Outputs | `out_RunId`, `Passed`, `SummaryMessage`. |
| Execution | The test is authored and validated but intentionally not run. |

## 2. Selection and safety rules

A blank `in_RunId` selects the newest record by `CreateTime` satisfying all of these conditions:

- `RunId` starts with `HIRE-`.
- `Email` ends with `@example.com`.
- `Stage` is `HireCompleted`, `HireSubmitted`, or `Failed` with a non-empty `HireEventRef`.
- The worker is not `Jane Doe`.

An explicit `in_RunId` uses the existing `GetHireRun.xaml` exact lookup, then the same eligibility gate is verified before any UI mutation. The workflow reads only the selected record's `FirstName`, `LastName`, `Email`, `EmployeeId`, `HireEventRef`, `PositionId`, `AtsCandidateId`, and `AtsApplicationId`. `ReservationRunId` and unrelated records are never changed.

## 3. Cleanup branches

### Rescind path — completed hire

For `HireCompleted`, the workflow verifies Logan McNeil is signed in and no proxy banner is present, searches Workday for the selected worker, verifies the employee ID, opens Job → Worker History, and verifies both the Hire and Propose Compensation Hire events. It reads the hire event status and requires `Successfully Completed`, then verifies the record stage agrees with `HireCompleted`.

It opens Related Actions → Business Process → Rescind, verifies the Rescind page, enters `UiPath demo cleanup ` plus the resolved RunId through the Chromium API input mode, verifies the page gate again, and submits. It verifies `Process Rescinded`, the rescinded event, event title suffix `(Rescinded)`, overall status `Rescinded`, position title suffix `(Unfilled)`, and incumbent count `0 items`.

### Cancel path — in-progress or failed hire

For `HireSubmitted` or eligible `Failed`, the workflow requires Workday status `In Progress`, then opens Related Actions → Business Process → Cancel. The existing Rescind Business Process OR controls are reused because the captured Cancel screen shares the same form controls. The comment, page gate, submit, event status, position title, and incumbent-count checks use those registered controls. This path was not live-proven on 14 Nov 2026; the assumption and limitation are recorded here rather than hidden behind placeholder targets.

## 4. ATS revert path

After Workday cleanup passes, the workflow opens `AtsBaseUrl & "/#/candidates/" & AtsCandidateId` in the existing ATS Candidate screen. It reads and verifies:

- ATS mode from the registered Jobs screen element: `Workday: LIVE (uipath_dpt1)`.
- Candidate name equals `FirstName & " " & LastName`.
- Current stage is `Hired`.
- Workday panel is present and identifies the hand-off.

It clicks the candidate's registered `Candidate_RevertHire_Button`, verifies the registered confirmation title is `Revert hire?`, gates the confirmation click on that result, and clicks the registered confirmation button. It then verifies the candidate is in `Offer`, the offer start date is nonempty, and the Workday panel contains `Not sent to Workday yet`.

API evidence uses the existing ATS endpoints without replacing UI actions:

- `GET /v1/candidates/{AtsCandidateId}` → application stage `Offer`, candidate status `active`, and `workday` null/empty.
- `GET /v1/events` → the newest `hire_reverted` event for the selected application has `payload.application_id = AtsApplicationId`.

## 5. Data Fabric persistence

Only after every Workday and ATS verification passes, the test invokes `Workflows/DataFabric/UpdateHireRun.xaml` with:

| Field | Value |
|---|---|
| `Stage` | `HireRunStage.CleanedUp` |
| `CleanupAction` | `HireCleanupAction.Rescind` or `HireCleanupAction.Cancel` |
| `HireEventStatus` | `Rescinded` or `Canceled` |
| `AtsReverted` | `True` |
| `CleanedUpAt` | `DateTimeOffset.Now` |
| `LastTestCase` | `TC-HR-HIRE-06` |

`UpdateHireRun.xaml` keeps its original arguments and adds optional nullable `in_CleanupAction`, `in_AtsReverted`, and `in_CleanedUpAt`. Each optional field is updated only when its argument has a value, preserving TC-HR-HIRE-01 through TC-HR-HIRE-04 callers.

## 6. Object Repository references

No targets were captured, resolved, renamed, replaced, or hand-written. Existing screen and element references are linked by reference.

### Workday

| Screen | Reference ID |
|---|---|
| Header | `e00i96boZ0Ke_iif9ZtTVw/yW1wjdzrNkuwGx3JfAwYug` |
| Search Results | `e00i96boZ0Ke_iif9ZtTVw/1J2bmEuijku09qAzkSCmOg` |
| Worker Profile | `e00i96boZ0Ke_iif9ZtTVw/hvEVhuoiTEi3MTdsg1K70Q` |
| Event Details | `e00i96boZ0Ke_iif9ZtTVw/vponzSyxgEe1ApZDahPgAg` |
| Rescind Business Process | `e00i96boZ0Ke_iif9ZtTVw/mF-LX89XLkipzP8qUm9DeA` |
| View Position | `e00i96boZ0Ke_iif9ZtTVw/bxQYgERxDUSXQnsIY1C6zg` |

Key element references: Header current user `3JKdvy2RMUO9LV-NPTFYrg`, global search `sVG5kcoypkSWzlVTXhXY_Q`, proxy banner `MR4YwDAmrU-bsb6a10jDHA`; employee result `pbA7UvJWsUWIC2TlAnJBnQ`, employee ID `06HOQRbXMkGXEHBcz_9-2g`; Job tab `hjfWX49KW0iuijxHjKqucQg`, Worker History `2vISV2hw9EShWl_W3WuvZg`, Hire event `dPyXbcduXUuPu-hk6j1dTQ`, Propose Compensation event `6Vo3cO_C8U6RWpcUR_H9EA`; event title `kFYa9LgJUkSsWppv2PMB0A`, overall status `n6g92hPlBE6SnLIgikafsA`, Details and Process `c11RQqAUP02kXOGQYTv8tQ`, Related Actions `KnXSiPJPw0iJCm1QUBhraQ`, menu item `qMLjdd6JzES4tseF4hUUhQ`, position `T9l-tt5kKkWMDGz4JQr9Mg`; Rescind page `Ft_wsgNAfUOu0v4AEUAEzQ`, comment `RpY8rdGjLECf_tnPw0P4aw`, submit `wKa6xYS0rE-lvdTfBdAPBg`, Process Rescinded `fngYrYmaGU2w2eGJ3elF6A`, rescinded event `ejeWmAa_z0ydIUCCPCxlYg`; position title `jPhiAfkts0iHnnjCZL6RRg`, incumbent count `POEK5NtD302M0Pq0TD8wlw`.

### TalentTrack ATS

Candidate screen `e00i96boZ0Ke_iif9ZtTVw/eqTq-EjRtU-OKkyjg1xfgA`; Jobs screen `e00i96boZ0Ke_iif9ZtTVw/_6dtvlP750WU67tvUQmFRg`.

Key element references: Jobs live mode `nmTuEBHy80qzUkrKeedcpg`; candidate name `7zR312fJtEypFz6XVUmUlQ`, current stage `a9ZJ46vin0-sObHx3RxuHg`, Workday panel `uB7trDGwVUanbdvnp0YnOA`, Workday status `KG-D7k0jYEyGp-JZvKZPZg`, revert hire `9TJ7egb0OEWCtQfeB89XwA`, confirmation title `9kuSujL9rUu6pU2JODf9Dw`, confirmation button `brTuS-7nHUq466iAQRCdbw`, Offer start date `wOkLH9P1ekuRVweAdlF5mQ`.

## 7. Verification inventory

Each listed expected result is implemented by one UiPath Testing Verify activity with `ContinueOnFailure=True`, success/failure screenshots enabled, and an explicit `CLN-01 step <n> - ...` title. String comparisons use `VerifyExpressionWithOperator`; Boolean conditions use `VerifyExpression`. UI gates are checked again before Workday submit and ATS confirmation.

## 8. Assumptions

- The existing OR screen definitions captured on 14 Nov 2026 remain valid and are the sole source for UI targets.
- The Worker Profile screen's Job and Worker History controls are sufficient to traverse the selected worker's profile; no separate Worker Job or Worker History screen is registered.
- The shared Rescind Business Process form controls are also valid for the unproven Cancel branch.
- The ATS candidate screen's URL-neutral selector can accept the dynamic candidate navigation performed by `NGoToUrl`.
- The ATS event API exposes event timestamps in a sortable `created_at` field; if absent, the newest matching event is still selected deterministically from the returned order.
- The test is not run because the only completed hire was already cleaned up, and running now would stop at the intended eligibility/state gates.
