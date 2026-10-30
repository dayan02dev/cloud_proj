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
| Trigger | committed in the Jenkinsfile: `triggers { pollSCM('* * * * *') }` (no UI trigger) |

## Builds

| Build | Cause | Commit | Result | Tests |
|---|---|---|---|---|
| #1 | Started by user admin (bootstrap run, loads the trigger) | `92c8a84` Configure my Jenkins assignment | SUCCESS | 28, 0 failures |
| #2 | **Started by an SCM change** (no Build Now) | `961c117` Personalize my help introduction | SUCCESS | 28, 0 failures |

Both runs went through every stage: Checkout → Submission setup → Test → Build JAR → Smoke test JAR → Save artifact.
In each run, the commit Jenkins printed at Checkout matches `git log -1 --oneline` for that commit.

## Red run (before the fix)

The fix (`33dedb8`) was committed before Jenkins was set up, so **no Jenkins red build
exists**. The original failure was recorded by running `java ci/Build.java test` on the
pre-fix commit `7a77072` (see `before-fix-local/test-output.txt`):

```
FAIL cancellationFreesSeat: SOLD OUT. Even the developer needs a ticket.
FAIL cancellationsSurviveReload: Cancellation was lost
FAIL cliWorkflowPersistsAcrossCommands: List does not reflect persisted cancellation
28 tests, 3 failures.
```

On a failed Test stage, the Jenkinsfile skips Build JAR, Smoke test JAR and Save artifact, so no JAR is published.
The `post { always }` block still archives `submission-evidence.csv`.

## Diagnosis and fix

**Failing assertion:** in `cancellationFreesSeat`, a 1-seat event is booked by S1, S1 cancels, then S2 books.
Expected: S2 is accepted and the roster is `[S2]`. Actual: `SOLD OUT`.

**Violated rule:** cancelling a booking must free the seat. `BookingService.cancel()` checked that
the booking existed but never removed the student, so a cancelled seat stayed taken
(and the cancellation was lost after save/reload).

**Fix (one line, `src/main/java/BookingService.java`):**

```java
event.students.remove(studentId);
```

The tests were not changed: the `FestPassTest.java` SHA-256 (`d203243b…a98a34`) is the same as in the
original starter. The only Jenkinsfile change is the 3-line `triggers` block.

## Final JAR

Downloaded from build #2 (`build-2/festpass.jar`, SHA-256 `079a16e9…01f5722`, which matches the CSV):

```
$ java -jar festpass.jar help
FestPass - campus event registration (personalized by dayan02dev)
Usage: java -jar build/festpass.jar [--data FILE] COMMAND
...
```

## Files

- `build-1/`: evidence CSV and console log for build #1
- `build-2/`: evidence CSV, `festpass.jar`, `build-info.txt`, `smoke-test.txt`, console log, `jar-help-output.txt`
- `before-fix-local/test-output.txt`: test run on the original buggy code
