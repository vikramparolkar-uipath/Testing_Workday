# TC-HR-HIRE-05 — NEG-01: Incomplete data is rejected

**Test Manager case:** `WDAY:312`  
**Workflow:** `Tests/TC-HR-HIRE-05.xaml`  
**Status:** Resolved and authored; per-file validation is clean; project build pending.  
**Execution status:** Not run by design. The test must not be executed until explicitly authorized.

## 1. Objective

Verify that:

1. Workday rejects a Create Pre-Hire form containing only first name and last name.
2. The failed Workday pre-hire is discarded and the generated negative identity is absent from Workday search results.
3. TalentTrack ATS creates a unique offerless candidate in `Application Review`.
4. TalentTrack rejects `POST /v1/applications/{applicationId}/hire` with HTTP `409` and the accepted-offer business error.
5. The candidate remains in `Application Review`, Mark as Hired remains disabled, no Workday hand-off exists, and no prohibited Workday webhook event exists.

## 2. Arguments and outputs

| Name | Direction | Type | Default or contract |
|---|---|---|---|
| `AtsBaseUrl` | In | `String` | `http://localhost:4100` |
| `FirstName` | In | `String` | Retained for the registered test-case contract; NEG-01 uses the fixed generated value `Neg`. |
| `LastName` | In | `String` | Retained for the registered test-case contract; NEG-01 uses the fixed generated value `Test` plus the suffix. |
| `WorkdayUiTimeout` | In | `TimeSpan` | `00:00:30` |
| `Passed` | Out | `Boolean` | True only when all verification results pass. |
| `SummaryMessage` | Out | `String` | Includes the run ID and failed verification titles when applicable. |
| `out_RunId` | Out | `String` | `"NEG-" & suffix` |

## 3. Generated test data

The workflow computes:

```vb
suffix = Now.ToString("ddMMyyHHmm", CultureInfo.InvariantCulture)
out_RunId = "NEG-" & suffix
```

Workday negative identity:

- First name: `Neg`
- Last name: `Test` & `suffix`
- No email, phone, or address is entered.

ATS offerless candidate:

- First name: `Neg`
- Last name: `Offerless` & `suffix`
- Email: `neg.offerless` & `suffix` & `@example.com`
- Phone: `+1 312 555 0142`
- Job: Accounting Specialist - AP, job ID `4001`

## 4. Object Repository references

No selectors were captured, re-resolved, renamed, replaced, or hand-authored for this workflow. All UI activities are linked to existing Object Repository entries.

### Workday

| Purpose | Reference ID |
|---|---|
| Workday Header screen | `e00i96boZ0Ke_iif9ZtTVw/yW1wjdzrNkuwGx3JfAwYug` |
| Workday Search Results screen | `e00i96boZ0Ke_iif9ZtTVw/1J2bmEuijku09qAzkSCmOg` |
| Workday Create Pre-Hire screen | `e00i96boZ0Ke_iif9ZtTVw/hxhR0qalIEa3b5KU-OPaoQ` |
| Current user button | `e00i96boZ0Ke_iif9ZtTVw/3JKdvy2RMUO9LV-NPTFYrg` |
| Global search input | `e00i96boZ0Ke_iif9ZtTVw/sVG5kcoypkSWzlVTXhXY_Q` |
| Dynamic search-result link; `resultText` is `Create Pre-Hire` or `Neg Test` plus suffix | `e00i96boZ0Ke_iif9ZtTVw/gvhI1tnqx0S-fwkBRqVk1g` |
| Create Pre-Hire first name | `e00i96boZ0Ke_iif9ZtTVw/wP_0rlx340anuSChT-yY5A` |
| Create Pre-Hire last name | `e00i96boZ0Ke_iif9ZtTVw/PP-H4CbDVk26ANSGkMiwpg` |
| Create Pre-Hire OK | `e00i96boZ0Ke_iif9ZtTVw/hSFW1NvixkG6SLj3vHVjlg` |
| Validation error | `e00i96boZ0Ke_iif9ZtTVw/cK5e6_0OrkO5V9ktj0D3xw` |
| Create Pre-Hire Cancel | `e00i96boZ0Ke_iif9ZtTVw/W4KqLa-EMUG4iJaNgHnfcw` |
| Discard Changes dialog label | `e00i96boZ0Ke_iif9ZtTVw/Y7-SCL8P6US4ozCkU0HGxg` |
| Discard Changes button | `e00i96boZ0Ke_iif9ZtTVw/S3T4peWDbE6iOseZERlcIQ` |

The generic Workday `No Results Message` reference is intentionally not used. Absence is proven with the dynamic result-link target.

### TalentTrack ATS

| Purpose | Reference ID |
|---|---|
| ATS Jobs screen | `e00i96boZ0Ke_iif9ZtTVw/_6dtvlP750WU67tvUQmFRg` |
| ATS Job screen | `e00i96boZ0Ke_iif9ZtTVw/9-Y8EvVxhUmS02saeGV7rQ` |
| ATS Candidate screen | `e00i96boZ0Ke_iif9ZtTVw/eqTq-EjRtU-OKkyjg1xfgA` |
| Workday live-mode text | `e00i96boZ0Ke_iif9ZtTVw/nmTuEBHy80qzUkrKeedcpg` |
| Job 4001 link | `e00i96boZ0Ke_iif9ZtTVw/v5lHGm-pv0GrOIh0ccMBWQ` |
| Add-candidate first name | `e00i96boZ0Ke_iif9ZtTVw/jUxRwZKBaEuVmBVkfdTyhA` |
| Add-candidate last name | `e00i96boZ0Ke_iif9ZtTVw/thV3SkK9A02gKHori_taJg` |
| Add-candidate email | `e00i96boZ0Ke_iif9ZtTVw/rFn6yF5OqkC98uUFHUvJcg` |
| Add-candidate phone | `e00i96boZ0Ke_iif9ZtTVw/DbZaR9TPxEixlclxp6xYTA` |
| Add-candidate submit | `e00i96boZ0Ke_iif9ZtTVw/0rsNObuLhkuTaEaN0ekVkQ` |
| Candidate name | `e00i96boZ0Ke_iif9ZtTVw/7zR312fJtEypFz6XVUmUlQ` |
| Current stage | `e00i96boZ0Ke_iif9ZtTVw/a9ZJ46vin0-sObHx3RxuHg` |
| Mark as Hired button | `e00i96boZ0Ke_iif9ZtTVw/jtjIxm-Y90CL5NDvlQnFMA` |
| Workday hand-off status | `e00i96boZ0Ke_iif9ZtTVw/KG-D7k0jYEyGp-JZvKZPZg` |

The Mark as Hired check uses the existing target's `disabled` attribute. The assertion expects `disabled = True`.

## 5. Given / When / Then sequence

### Given — session and generated identity

1. Generate the invariant-culture suffix and expose `out_RunId`.
2. Invoke the existing `Workflows/Workday/CheckIfSignedIn.xaml` with expected user `Logan McNeil`.
3. Verify the signed-in precondition. If it fails, stop before mutating either application.

### When — Workday negative pre-hire

4. Use the Workday global search with `Simulate` text input and a separate Hardware Events Enter shortcut.
5. Open the `Create Pre-Hire` result with a `Simulate` click.
6. Enter only `Neg` and `Test` plus suffix with `Simulate` Type Into activities.
7. Click OK with `Simulate` and read the combined Workday error text.
8. Verify that the error contains:

   `Please enter at least one address, phone number, or email address.`

9. Click Cancel. Check the existing Discard Changes dialog and click Discard when it appears. Verify that the discard dialog was detected and the incomplete form was discarded.
10. Search Workday for the generated negative identity using `Simulate` text input and a separate Hardware Events Enter shortcut.
11. On the Search Results screen, use Check App State against the dynamic result-link target with `resultText = "Neg Test" & suffix`. Verify `Exists = False`. The generic no-results card is not used as evidence.

### When — ATS offerless candidate and rejected hire

12. Verify `Workday: LIVE (uipath_dpt1)` on the ATS Jobs screen.
13. Open job `4001`, Accounting Specialist - AP, with a `Simulate` click.
14. Create the unique candidate with `Simulate` Type Into/Click activities.
15. Verify the candidate name, `Application Review` stage, `disabled = True` for Mark as Hired, and Workday text containing `Not sent to Workday yet`.
16. Gate the API call: if ATS live mode or Mark as Hired disabled verification failed, throw before the POST.
17. GET `{AtsBaseUrl}/v1/candidates` and deserialize the JSON array.
18. Find the candidate by `emails[0].value`, then extract `id` and `applications[0].id`.
19. POST `{AtsBaseUrl}/v1/applications/{applicationId}/hire` with:

   - Method: `POST`
   - `RequestBodyType`: `Text`
   - `TextPayload`: `{}`
   - `FormDataParts`: `Nothing`
   - Expected status: `409 Conflict`

20. Verify the response body contains `An accepted offer is required before hiring`.
21. GET `/v1/candidates/{candidateId}` and verify:

   - Application stage is `Application Review`.
   - Candidate status is not `hired`.
   - `workday` is absent or empty.

22. GET `/v1/events`, newest first, and verify no event for the current application has action `workday_prehire_synced` or `candidate_hired`.
23. Refresh the ATS candidate page with Hardware Events F5 and repeat the UI checks for `Application Review` and `Not sent to Workday yet`.

### Then — result aggregation

24. Every verification uses `ContinueOnFailure=True`, success/failure screenshots, an explicit title, and an explicit output message format.
25. Aggregate all verification results into `Passed`.
26. Build `SummaryMessage` and throw once at the end if any verification result is false, including failed verification titles.

## 6. Verification inventory

| Step | Verification |
|---:|---|
| 1 | Workday is signed in as Logan McNeil. |
| 2 | Workday rejects missing contact information. |
| 3 | Workday incomplete form is discarded. |
| 4 | Workday rejected candidate is absent from the dynamic result link. |
| 5 | ATS Workday integration is live. |
| 6 | ATS creates the unique offerless candidate. |
| 7 | ATS candidate is in Application Review. |
| 8 | Mark as Hired disabled attribute is true before the API call. |
| 9 | ATS Workday hand-off text says it has not been sent. |
| 10 | ATS API finds the candidate by unique email. |
| 11 | ATS hire endpoint returns HTTP 409. |
| 12 | ATS requires an accepted offer before hiring. |
| 13 | API candidate remains in Application Review. |
| 14 | API candidate is not hired. |
| 15 | API candidate has no Workday hand-off. |
| 16 | No prohibited Workday webhook event exists for the application. |
| 17 | Refreshed ATS candidate remains in Application Review. |
| 18 | Refreshed ATS candidate still has no Workday hand-off. |

## 7. Assumptions

- The existing Workday browser session is already signed in as Logan McNeil; the reusable check verifies this and no password or sign-in action is included.
- Existing browser tabs and the captured OR screen scopes remain available; no new window, proxy, or browser session is started.
- `disabled` is the authoritative attribute for the captured Mark as Hired button, and the live negative candidate exposes it as `True`.
- Workday's captured error target returns combined visible text, so the expected message is checked with `Contains`.
- Workday absence is determined only from the dynamic result-link target, because unrelated task/report cards can coexist with the generic no-results message.
- The ATS mock API returns candidates as a JSON array and exposes `emails[0].value`, `id`, and `applications[0].id`.
- The ATS hire endpoint accepts an empty JSON object and returns the expected business rejection without changing candidate state.
- The ATS event endpoint returns newest events first and exposes event action plus `payload.applicationId`.
- No Data Fabric entity is queried or written, and no `WorkdayHireRun` record is created.
- No Try/Catch, Retry Scope, fixed Delay, offer recording, Mark as Hired action, reset-demo action, or unrelated ATS mutation is included.

## 8. Validation and build

- `uip rpa validate --file-path Tests/TC-HR-HIRE-05.xaml --project-dir <project> --output json`: **clean; no diagnostics found.**
- Object Repository screen linking: **9 screens linked successfully.**
- Object Repository element linking: **all required element links attached successfully.**
- `uip rpa build <project>`: **passed** with `Data.Success: true`.
- Build warnings: the known unrelated Data Service generic-constraint warnings for `TC-HR-HIRE-01.xaml` and related `WorkdayHireRun` workflows, plus the environment warning that `dotnet` CLI is unavailable for compiler/package-signing features. No TC-HR-HIRE-05 build error was reported.
- Test execution: **intentionally not run**; explicit user authorization is required.

## 9. Scope boundary

This change does not modify or run `TC-HR-HIRE-01.xaml`, does not publish to Orchestrator, and does not modify the existing evidence candidate or any unrelated ATS record.
