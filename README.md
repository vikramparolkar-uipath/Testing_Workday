# Testing_Workday — UiPath Test Cloud demo for Workday HCM

A UiPath Studio test automation project that tests four Workday HCM business processes end to end:
Hire, Termination, Change Compensation and Change Job. It is a **cross-platform** project
(Studio 26.10, Portable / .NET, VB.NET expressions), so the same tests run on a local robot or on
Test Cloud serverless robots in Chrome.

> The previous Windows-only version of this repository (`TC_*.xaml` files, `TestFramework`,
> `z_GoldenDemo`) was replaced by this project in November 2026.

## What's in the box

| Test set | Test cases | What it proves |
|---|---|---|
| Hire (integration) | `TC-HR-HIRE-01` … `06` | A candidate is marked hired in a mock ATS (TalentTrack), the pre-hire lands in Workday, the hiring manager starts Hire Employee, starting pay is proposed and approved, a negative test rejects incomplete data, and the run cleans up after itself. The six tests run as a chain that shares one run record. |
| Termination | `TC-HR-TERM-01` … `06` | Record the worker's job before a termination, a voluntary termination that the HR partner approves, one the HR partner denies, an involuntary termination that is then rescinded, a missing Termination Date that is refused, and a rescind that resets the worker. |
| Change Compensation | `TC-HR-CC-01` … `05` | Record the worker's pay and pay range, a 5% raise through the full approval chain, a 20% raise that adds Controller approval (through proxy), a change that is sent back, and a cancel that requires a comment. |
| Change Job | `TC-HR-CJ-01` … `07` | Record the worker's job, a business title change, a promotion to Controller with a raise through the full approval chain, a promotion that is denied, a team transfer that adds the receiving manager's review, a wizard that won't proceed without required fields, and a completed promotion that is rescinded. |

Each test case comes in two versions:

- `Tests/<set>/TC-….xaml` is the plain test case.
- `Tests/<set>/With Execution Template/TC-…-ET.xaml` is the same test with the
  `.templates/Workday Test Execution.xaml` execution template attached. These are the versions
  linked to Test Manager and run from Test Cloud.

The Compensation and Change Job tests are **independent**. Each one starts from a clean state for
its worker (`Workflows/Workday/CleanStartForWorker.xaml` cancels in-progress events and rescinds
leftover future-dated promotions), creates its own data and cleans up what it created, so they can
run in any order.

The Termination tests share one worker and run in a fixed order: 01, 02, 06, 03, 04, 05.
TC-HR-TERM-02 needs no termination pending, and TC-HR-TERM-06 rescinds the completed termination so
the later tests can run. The three sets can run in parallel with each other.

## Project layout

```text
Tests/                     Test cases (Hire at the root, plus ChangeJob/, Compensation/, Termination/)
Workflows/Workday/         Reusable Workday steps: sign in/out, proxy, search, inbox, worker history
  ChangeJob/               Change Job wizard steps and promotion approvals
  Compensation/            Request, review, submit and cancel compensation changes
  Termination/             Terminate Employee steps
Workflows/DataFabric/      Read and update the Hire run record (WorkdayHireRun entity)
.templates/                Execution template and test case templates (UI, ATS API)
.objects/, .screenshots/   Object Repository for Workday and TalentTrack ATS
.entities/                 Data Fabric entity definitions used by the tests
specs/                     Written test specifications for the Hire set
AGENTS.md, CLAUDE.md       Instructions for coding agents working on this project
```

## How it works

- **Credentials** are never stored in the project. `Workflows/Workday/SignInAs.xaml` and
  `Login to Workday.xaml` read Orchestrator credential assets (folder `Workday`) by name, one per
  Workday persona (manager, HR partner, and so on).
- **Test data** comes from UiPath Data Fabric. The Hire chain passes its state between tests in
  the `WorkdayHireRun` entity. The other sets can read a worker assignment from
  `WorkdayTestReservation`, written by the Workday Test Data Steward agent, and otherwise fall back
  to their default workers.
- **Targeting** uses Modern UI Automation with Object Repository selectors. Steps that are fragile
  in Workday, such as saving an inline salary row, use ScreenPlay (UI Task) as the primary method,
  with selector checks before and after.
- **Cloud robots** start each test with Chrome as the runtime browser and end with
  `Workflows/Workday/CloseWorkday.xaml` (sign out and close). Steps that change the page are written
  check-then-click with read-back, because serverless robots act faster than Workday re-renders.

## Running the tests

Prerequisites:

1. UiPath Studio 26.10 or later, or the `uip` CLI, signed in to your Test Cloud tenant.
2. An Orchestrator folder `Workday` with credential assets for each persona the tests sign in as.
3. Access to a Workday tenant with the demo workers the tests use.
4. For the Hire set only: the TalentTrack mock ATS running on `http://localhost:4100`. A cloud
   robot can't reach localhost, so the Hire set runs on a local robot.

Run one test from the CLI:

```bash
uip rpa run --file-path "Tests/Termination/TC-HR-TERM-01.xaml" --project-dir . --timeout 1200
```

Run the Hire chain in order. `TC-HR-HIRE-01` returns `out_RunId`. Pass it to tests 02, 03, 04 and 06
with `--input-arguments '{"in_RunId":"<RunId>"}'`. Without it, each test picks up the newest run at
its stage. `TC-HR-HIRE-05` is the negative test and creates its own run.

In Test Manager, the `-ET` entry points are linked to test cases and grouped into four test sets
(`TS_Hire`, `TS_Termination`, `TS_Compensation`, `TS_ChangeJob`) with the run order enforced. On
Test Cloud serverless robots, the Termination, Compensation and Change Job sets have passed 18 out
of 18, both back to back and in parallel (about 22 minutes in parallel).

## Working on the project with coding agents

The project was built and is maintained with coding agents (Claude Code with the UiPath `uip` CLI
and skills, and UiPath Autopilot). `AGENTS.md` holds the house rules. In short:

- Run `uip rpa validate` and `uip rpa build` after every change.
- Leave Studio's view-state blocks alone.
- Don't commit `.local/`.
- Error handling belongs to the execution template, not to the test case.
- Verifications continue on failure, with screenshots and clear output messages.

## Version

Project version **1.0.25**, published to the Test Cloud tenant feed.
