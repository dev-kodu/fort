# Logging & Diagnostics Standard

**MANDATORY** on every risky path in this repo. Adapted to *this* codebase — Qt categorized logging
in the UI/service, I/O-error-log events in the kernel driver.

## Why: how this thing is actually debugged

Fort runs as a **Windows service + a separate UI process + a kernel-mode WFP driver** on a user's
machine. You will not have a debugger attached when it matters. A kernel fault is a **BSOD** — the
process is gone, and nothing you write after the fault survives. The log is the whole story.

**Logging must be verbose enough to diagnose a wrong result — or a crash — from the log file alone.**

## The mechanics you must know (verified in this repo)

### UI / service side — Qt categorized logging

Per-`.cpp`, in an anonymous namespace at the top (this is the established idiom — follow it):

```cpp
namespace {
const QLoggingCategory LC("driver.manager");   // dotted, subsystem.component
}
...
qCDebug(LC) << "executeCommand:" << cmdPath << scriptPath << fileName;
```

Two facts from `src/ui/manager/logger.cpp` that change how you must write log lines:

1. **`qCDebug` does NOT reach the log file unless debug logging is enabled.**
   `Logger::processMessage` computes `isLogToFile = (level != Logger::Debug || logger->debug())`.
   In a default install, **every `qCDebug` line is invisible.** `qCInfo` / `qCWarning` /
   `qCCritical` always reach the file.
   > **Corollary — this is the rule that bites:** a diagnostic that only matters when something is
   > about to go wrong (see *the begin-log rule*) must **not** be `qCDebug`. If it is, the one line
   > you needed was never written, and you will not know it was missing.
2. **Every line is flushed as it is written.** `Logger::writeLogLine` calls `m_file.flush()` after
   each successful write. So the "flush immediately" requirement is satisfied by the existing
   logger — **provided the line was emitted at all** (see fact 1). Do not add buffering.

Log file: directory from the `--logs` option or the `global/logsDir` setting. Rotation is
**1 MiB per file, 9 files kept** (`LOGGER_FILE_MAX_SIZE`, `LOGGER_KEEP_FILES`). Budget accordingly:
a chatty `qCDebug` in a hot loop will **evict the evidence you need**. Verbosity is for risky paths,
not for everything.

UI crashes additionally produce **Breakpad minidumps** (`src/ui_bin/3rdparty/breakpad/`). A minidump
tells you *where* it died; only the log tells you *what it was doing*. You need both.

### Kernel side — `TRACE()`

`src/driver/forttrace.h`:

```c
TRACE(event_code, status, error_value, sequence);   // -> fort_trace_event()
```

This is **not ETW and not a log file**. `fort_trace_event` (`forttrace.c`) calls
`IoAllocateErrorLogEntry` / `IoWriteErrorLogEntry` — the **I/O error log**, which surfaces in the
**Windows System event log**. Event codes are defined in `src/driver/evt/fortevt.mc` (message
compiler → `FORTEVT_MSG00001.bin`). You get **four numbers** — no strings, no formatting. **Design
event codes so the four numbers identify the culprit**; an event that only says "the callout failed"
is useless.

**Two silent-drop conditions you must design around — read `forttrace.c` before relying on TRACE:**

```c
if (KeGetCurrentIrql() > DISPATCH_LEVEL || fort_device() == NULL)
    return;                       /* (1) trace silently does nothing */
PIO_ERROR_LOG_PACKET packet = IoAllocateErrorLogEntry(...);
if (packet == NULL)
    return;                       /* (2) dropped under memory pressure */
```

1. **Above `DISPATCH_LEVEL`, `TRACE()` is a no-op.** The hottest, most dangerous paths are exactly
   where it goes silent. Do not assume an absent event means the code did not run.
2. **It is dropped when the pool is exhausted** — i.e. *precisely* during the OOM you are trying to
   diagnose. `TRACE(FORT_BUFFER_OOM, ...)` can itself be lost to OOM.

This is the kernel counterpart of the `qCDebug` trap: **an absent line is not evidence of absence.**
It is also why prevention beats tracing down here — see the begin-log rule.

`FORT_CHECK_STACK(func_id)` (`fortdbg.h`, behind `FORT_DEBUG_STACK`) exists for stack checking.

## The begin-log rule — log BEFORE the thing that can kill you

**Log before a risky step, at the granularity of the thing that can kill you.**

- **Per item, before the risky call — not per phase.** A single line before an N-item loop is
  **NOT compliant**. When it dies, "begin phase" names the *phase*; you need the *item*.
- **The line must actually be emitted and flushed** → `qCInfo`/`qCWarning`, never `qCDebug`
  (see above). Qt's handler + `Logger` flush per line; do not defeat that.
- **Include a thread id** when the risky loop is parallel — `WorkerManager` runs up to
  `maxWorkersCount()` workers over one job queue, so interleaved lines are otherwise unattributable.
- **Where failure is uncatchable, a `try`/`catch` documented "best-effort, never throws" is a LIE.
  Prevent, do not catch.** In `src/driver/` this is literal: you cannot catch a bugcheck. Validate
  lengths, check IRQL, check the lock, check the pointer — **before** you dereference.

### The evidence (why this rule exists, earned the hard way)

In a sibling repo, `XbimGeometrySidecar` logged exactly **one** line — `begin CreateContext()` — then
pushed **583,129 entities** through native code and died with **exit 139**. It was
**nondeterministic: it survived 1 run in 3.** The last log line named the **phase**, not the
**culprit**, so three runs produced three identical, useless logs.

> **"IF ITS NONDETERMINISTIC YOU DONT LOG ENOUGH."**

A nondeterministic crash is not a mystery — it is a **logging defect**. If you cannot name the item
that killed you, add the per-item begin line and run it again.

**In this repo the same shape is everywhere:** `ConfBuffer::writeConf` walks every app and every
rule into a packed blob; `fort_pstree_update_services` walks a service list **in kernel memory**;
`fortdl`/`fortmm` maps a driver image. Each is an N-item loop whose failure mode is silent
corruption or a bugcheck. One "begin writeConf" line before the loop tells you nothing.

## The risky paths in *this* repo — and what each must log

### 1. The UI→driver config blob (`src/ui/util/conf/`) — highest risk
`ConfBuffer` / `ConfData` build a **packed binary struct** that **kernel code parses**. A wrong
offset or length is a bugcheck or silent misfiltering.
Log: blob **byte size** and each section's **offset + length**; counts of apps / groups / rules /
zones written; `DRIVER_VERSION` written vs expected; every value that crossed a **unit conversion**
(e.g. `writeLimitBps`: kbits/s in → bytes/s out — log both); and `errorMessage()` on failure.
**Per item before the write**, not once before the loop.

### 2. The IOCTL boundary (`Device::ioctl`, `src/ui/driver/driverworker.cpp`)
Log: IOCTL code, `inSize`, `outSize`, returned size, and the **Win32 error** on failure. A mismatch
between `inSize` and what the driver expects is the classic version-skew bug — the returned
`STATUS_UNSUCCESSFUL` from `fort_device_control_validate` means *nothing* without the sizes.

### 3. Kernel driver (`src/driver/`)
`TRACE()` before the risky operation, with a code that identifies **which** callout/flow/packet.
Log allocation failures (`FORT_BUFFER_OOM` is the existing pattern), registration failures with
their `NTSTATUS`, and flow-association errors. Bound every loop and every recursion. Never trust a
length that came from user mode — `dca->in_len` is attacker-adjacent, and
`fort_pstree_update_services` walks it with `while (n-- > 0)` **in kernel memory**.

**But remember `TRACE()` is a no-op above `DISPATCH_LEVEL` and is dropped on OOM.** Down here the
standard's real requirement is not "log more", it is **prevent**: validate the length, assert the
IRQL, take the lock, check the pointer — *before* the dereference. You will not get a second chance
and you will not get a log line.

### 4. Path / rule matching (`ConfUtil`, `ruletextparser`, `3rdparty/wildmatch`)
**A wrong answer here silently allows traffic that should be blocked.** Log: the **input path**, the
parsed result, `isWild`/`isPrefix`, and **which rule matched** (id) for a decision. Without the input
string in the log, a user's "it didn't block X" is unreproducible.

### 5. SQLite + migrations (`src/ui/3rdparty/sqlite/`, `*/migrations/*.sql`)
Log **before** each migration: current user_version → target, and the file about to run. Log the
SQL error text on failure. This runs against **irreplaceable user data** (traffic history). A
migration that dies mid-way must leave a log line naming the exact step.

### 6. Workers / job queues (`src/ui/util/worker/`, `task/`, `stat/`)
Log per job: **thread id**, job type/id, enqueue → start → finish, and elapsed ms. `WorkerManager`
has a `WORKER_TIMEOUT_MSEC` (5000) dequeue timeout and an abort path — log aborts and timeouts
explicitly; a silently aborted worker looks identical to one that never ran.

### 7. Network (`NetDownloader`, `taskzonedownloader`, `autoupdatemanager`)
Log: URL, HTTP status, bytes received, and **each retry with its attempt number**. **Bound retries**
— an unbounded retry against a dead zone URL is an infinite loop on a user's machine.

### 8. Service ↔ UI RPC (`src/ui/rpc/`, `src/ui/control/`)
Log both sides of the boundary: command, arguments, and the result. "The UI showed nothing" is
otherwise indistinguishable from "the service never got it".

## Timing

Time each phase with `QElapsedTimer` and emit a completion line:

```
DONE writeConf in 42ms
```

`DONE <what> in Nms` — for conf builds, driver reloads, DB migrations, zone downloads, and worker
jobs. A frozen UI or a slow startup must be attributable to a **named phase**, not guessed at.

## Logging is best-effort and must NEVER break the workflow

- A logging failure must never propagate. `Logger` already swallows write failures; keep it that
  way. Never let a log line throw, block, or change control flow.
- **Never log secrets.** The password dialog (`form/dialog/passworddialog.cpp`) and RPC credentials
  are off limits.
- **Never log in a hot packet path at `qCDebug` volume** — 1 MiB × 9 rotation will erase the
  evidence, and in the driver you cannot afford the IRQL cost.
- Do not add a logging framework. Use `qCInfo`/`qCWarning`/`qCDebug` with a category, and `TRACE()`
  in the driver. That is what exists and what the log file understands.

## File-level logging review on every edit (MANDATORY)

When you edit a **pre-existing** file, review the **whole file's** logging — not just your lines —
and bring it to this bar in the same change.

Checklist for the file you touch:
- Risky paths log inputs, per-item begin lines, counts, sizes/offsets, units, and outcomes.
- Begin lines are `qCInfo`/`qCWarning` (not `qCDebug`), emitted per item, before the risky call.
- Parallel loops log a **thread id**; phases emit `DONE ... in Nms`.
- Retries and recursion are **bounded**; aborts and timeouts are logged explicitly.
- Logging cannot throw, block, or leak secrets.

**Keep it proportional.** This is logging hygiene on a file you were already editing — not a licence
to refactor, and not permission to churn upstream code in a zero-divergence fork. If the file is
genuinely at the bar, change nothing — but you must have looked.
