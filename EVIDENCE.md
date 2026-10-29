# FestPass CI Assessment Evidence

**Student Alias:** shubhk2  
**Class Code:** 001  
**Challenge Code:** 00824  
**Job Name:** festpass-ci  
**Date:** 2026-10-29  

---

## 1. Build Trace & Commits

| Build # | Trigger / Cause | Commit ID | Commit Message | Result | Test Results | Published Products |
|:---:|:---|:---|:---|:---:|:---:|:---|
| **1** | Started by user `shubhk2` (Manual bootstrap) | `b2c52a5` | Configure my Jenkins assignment | **FAILURE** | 25 passed, 3 failed | `submission-evidence.csv` *(JAR gated)* |
| **2** | **Started by an SCM change** (Automatic polling) | `e728b2d` | Personalize my help introduction | **FAILURE** | 25 passed, 3 failed | `submission-evidence.csv` *(JAR gated)* |
| **3** | **Started by an SCM change** (Automatic polling) | `551a1f6` | Fix seat cancellation by removing student from registration roster | **SUCCESS** | **28 passed, 0 failed** | `festpass.jar`, `build-info.txt`, `smoke-test.txt`, `submission-evidence.csv` |

---

## 2. Failing Assertion & Diagnosis

### Observed Failure
In Build 1 and Build 2, the `Test` stage failed with 3 test failures:
- `cancellationFreesSeat`: `java.lang.IllegalArgumentException: SOLD OUT. Even the developer needs a ticket.` at `BookingService.book(BookingService.java:60)`
- `cancellationsSurviveReload`: `java.lang.AssertionError: Cancellation was lost` at `FestPassTest.check(FestPassTest.java:343)`
- `cliWorkflowPersistsAcrossCommands`: `java.lang.AssertionError: List does not reflect persisted cancellation` at `FestPassTest.check(FestPassTest.java:343)`

### Violated Business Rule
When a student cancels their registration for an event, that seat must be freed up and returned to available capacity, allowing other students to book it. Furthermore, the cancellation must be reflected on the event roster and saved into persistent storage.

In `src/main/java/BookingService.java`, the `cancel(String eventId, String studentId)` method validated that the student was booked, but never actually modified the `event.students` collection:
```java
// BEFORE (Buggy):
public void cancel(String eventId, String studentId) {
    validateId(studentId);
    Registration event = requireEvent(eventId);
    if (!event.students.contains(studentId)) {
        throw new IllegalArgumentException("No booking for student: " + studentId);
    }
    // Missing: removal of student from event.students!
}
```

### The Minimal Fix
Added `event.students.remove(studentId);` to remove the student ID from the internal set:
```java
// AFTER (Fixed):
public void cancel(String eventId, String studentId) {
    validateId(studentId);
    Registration event = requireEvent(eventId);
    if (!event.students.contains(studentId)) {
        throw new IllegalArgumentException("No booking for student: " + studentId);
    }
    event.students.remove(studentId);
}
```

---

## 3. Pipeline Gating & Artifact Behavior

- **On Failure (Builds 1 & 2):**
  The `Test` stage failed, causing Jenkins to immediately skip the downstream product stages:
  - `Build JAR`: Skipped
  - `Smoke test JAR`: Skipped
  - `Save artifact`: Skipped
  No `festpass.jar` was created or archived. Diagnostic `submission-evidence.csv` was archived in the `always` post block.

- **On Success (Build 3):**
  All 28 unit tests passed. Downstream stages executed cleanly:
  - `Build JAR`: Compiled and packaged `build/festpass.jar`
  - `Smoke test JAR`: Successfully ran all 7 live CLI assertions against the packaged JAR
  - `Save artifact`: Archived `build/festpass.jar`, `build/build-info.txt`, `build/smoke-test.txt`, and `build/submission-evidence.csv`.

---

## 4. Final JAR Execution Output

Running `java -jar festpass.jar help` from the download directory:

```text
$ java -jar festpass.jar help
FestPass - campus event registration (shubhk2)
Usage: java -jar build/festpass.jar [--data FILE] COMMAND
  add-event ID "TITLE" CAPACITY
  list
  book EVENT_ID STUDENT_ID
  cancel EVENT_ID STUDENT_ID
  roster EVENT_ID
  summary
  help
Default data file: data/festpass.tsv
IDs are case-sensitive. Use synthetic student IDs, not personal data.
Exit codes: 0 success, 2 invalid request, 3 storage failure.
```

Full CLI verification workflow:
```text
$ java -jar festpass.jar --data /tmp/demo.tsv add-event java "Java Workshop" 1
EVENT CREATED: java

$ java -jar festpass.jar --data /tmp/demo.tsv book java S1
BOOKING ACCEPTED: S1 -> java

$ java -jar festpass.jar --data /tmp/demo.tsv book java S2
Invalid input: SOLD OUT. Even the developer needs a ticket.

$ java -jar festpass.jar --data /tmp/demo.tsv cancel java S1
BOOKING CANCELLED: S1 -> java

$ java -jar festpass.jar --data /tmp/demo.tsv book java S2
BOOKING ACCEPTED: S2 -> java

$ java -jar festpass.jar --data /tmp/demo.tsv summary
Events: 1 | Booked: 1 | Available: 0
```
