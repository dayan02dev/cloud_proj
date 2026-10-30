# FestPass Jenkins assignment: evidence

- **Project:** FestPass (`festpass-java`)
- **Jenkins job:** `festpass-ci`
- **Git author alias:** `dayan02dev`

> **Issued codes:** `submission.properties` currently holds the placeholders
> `REPLACE-ME-CLASS` / `REPLACE-ME-CHALLENGE`. Replace them with the
> instructor-issued codes, then commit. Polling will run a new automatic build,
> and its CSV becomes the graded green evidence.

## Job configuration

| Setting | Value |
|---|---|
| Type | Pipeline, *Pipeline script from SCM* → Git |
| Repository | `git://127.0.0.1:9418/festpass-java` (local read-only git daemon) |
| Credentials / branch / script path | none / `*/main` / `Jenkinsfile` |
| Agent | `student-laptop` (inbound WebSocket agent; built-in node has 0 executors) |
| Trigger | committed in the Jenkinsfile: `triggers { pollSCM('* * * * *') }` (no UI trigger, no "Build periodically") |

## Graded builds: automatic red, then automatic green

| Build | Cause | Commit | Result | Tests | JAR |
|---|---|---|---|---|---|
| **#5 (red)** | Started by an SCM change | `50c55ac` Revert cancellation fix to reproduce the seeded failure in Jenkins | **FAILURE** | 28, **3 failures** | not published (`NOT_PUBLISHED`) |
| **#6 (green)** | Started by an SCM change | `cc7ca16` Restore cancellation fix: free the seat when a booking is cancelled | **SUCCESS** | **28, 0 failures** | `festpass.jar` |

Neither build was started with Build Now. In each run, the commit Jenkins printed at Checkout matches `git log -1 --oneline` for that commit.

**Red run #5, stages:** Checkout ✓ → Submission setup ✓ → Test ✗ → then
`Stage "Build JAR" skipped due to earlier failure(s)`,
`Stage "Smoke test JAR" skipped due to earlier failure(s)`,
`Stage "Save artifact" skipped due to earlier failure(s)`.
Only `submission-evidence.csv` was archived. The JAR URL for #5 returns 404.

**Green run #6, stages:** Checkout → Submission setup → Test (28/0) → Build JAR → Smoke test JAR (seven real-JAR commands pass) → Save artifact.

### Why the history has a revert

I fixed the bug (`33dedb8`) before Jenkins was set up, so the first builds
(#1 manual bootstrap, then #2–#4 automatic) were already green. To record the required automatic
red run, `50c55ac` reverts that fix. This restores the original starter `BookingService.java`
byte for byte; no new defect is introduced. `cc7ca16` then re-applies the fix. Build #4 was
triggered by a local commit (`8236e74`) that I amended by mistake and dropped before pushing;
its file contents were identical to `06ff85a`.

## Diagnosis and fix

**Failing assertions (build #5 test report):**

```
cancellationFreesSeat              IllegalArgumentException: SOLD OUT. Even the developer needs a ticket.
cancellationsSurviveReload         AssertionError: Cancellation was lost
cliWorkflowPersistsAcrossCommands  AssertionError: List does not reflect persisted cancellation
```

In `cancellationFreesSeat`, a 1-seat event is booked by S1, S1 cancels, then S2 books.
Expected: S2 is accepted and the roster is `[S2]`. Actual: `SOLD OUT`.

**Violated rule:** cancelling a booking must free the seat. `BookingService.cancel()` checked that
the booking existed but never removed the student, so a cancelled seat stayed taken
(and the cancellation was lost after save/reload).

**Fix (one line, `src/main/java/BookingService.java`):**

```java
event.students.remove(studentId);
```

Before committing, `java ci/Build.java verify` gave 28 tests, 0 failures, and `java ci/SmokeTest.java` passed all seven commands.
The tests were not changed: the `FestPassTest.java` SHA-256 (`d203243b…a98a34`) is the same as in the original
starter and is identical in both CSVs. The only Jenkinsfile change is the 3-line `triggers` block.

## Final JAR

Downloaded from build #6 (`build-6-green/festpass.jar`, SHA-256 `36b1cd2f…ef87a8af1`, which matches the green CSV):

```
$ java -jar festpass.jar help
FestPass - campus event registration (personalized by dayan02dev)
Usage: java -jar build/festpass.jar [--data FILE] COMMAND
...
```

## Files

- `build-5-red/`: red evidence CSV and console log
- `build-6-green/`: green evidence CSV, `festpass.jar`, `build-info.txt`, `smoke-test.txt`, console log, `jar-help-output.txt`
- `before-fix-local/test-output.txt`: local test run on the original starter code
