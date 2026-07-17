# Code Comments & Documentation Standard

**MANDATORY** for all code written or edited in this repo. Adapted to *this* codebase (C++20/Qt +
C kernel driver). Do not import rules from other repos' toolchains.

## Scope — read this first, it keeps the rule honest

This codebase has **zero doc comments today**: of the **283** first-party headers under `src/ui/`
(excluding vendored `3rdparty/`), **0** contain `///` or `/**`, and there is no Doxygen setup. That
is the starting point, and it is not a backlog to burn down.

- **This standard applies to code you write or change.** New public API is documented at
  write-time.
- **It is NOT a license to retrofit docs onto 998 existing files.** Do not open a PR that documents
  code you did not otherwise touch. This is a fork at zero divergence from `upstream/master`; every
  cosmetic line we add is a future merge conflict for no benefit (see `CLAUDE.md` → Fork discipline).
- **Never document vendored code.** `src/3rdparty/`, `src/ui/3rdparty/` are upstream's.

A rule nobody follows is worse than no rule. This one is small on purpose.

## The bar: what a doc comment must say

A doc comment states the **effect**, the **default**, the **meaning of edge values**, and the
**units** — never a restatement of the name.

> `// Returns the speed limit.` on `speedLimitIn()` is worthless. It says nothing the name did not.

**This repo already contains the exemplar.** `src/driver/common/fortconf.h`:

```c
typedef struct fort_speed_limit
{
    UINT16 plr; /* packet loss rate in 1/100% (0-10000, i.e. 10% packet loss = 1000) */
    UINT32 latency_ms; /* latency in milliseconds */
    UINT32 buffer_bytes; /* size of packet buffer in bytes (150,000 is the dummynet's default) */
    UINT64 bps; /* bandwidth in bytes per second */
} FORT_SPEED_LIMIT, *PFORT_SPEED_LIMIT;
```

Every field carries its unit, its scale, its edge values, and a default. **Match this bar.**

### Units are not optional

This is a firewall: almost every number has a unit, and getting it wrong is a real bug. Real,
current example of the gap — `src/ui/conf/appgroup.h`:

```cpp
quint32 speedLimitIn() const { return m_speedLimitIn; }   // unit: not stated anywhere
```

To learn the unit you must find `src/ui/util/conf/confdata.cpp:39`:

```cpp
void writeLimitBps(PFORT_SPEED_LIMIT limit, quint32 kBits)
{
    limit->bps = quint64(kBits) * (1024LL / 8); /* to bytes per second */
}
```

So `speedLimitIn()` is **kilobits/s**, and the C++→C boundary converts it to **bytes/s** using
**1024**, not 1000. Three files to answer "what unit is this?" — that is the cost this standard
removes. Units in play here: bytes, bytes/s, kbits/s, KiB vs kB (`src/ui/util/formatutil.h`
distinguishes both), ms, 1/100 %, ports, IPv4/IPv6 ranges, seconds.

State the unit **at the declaration**, in the header, where the caller reads it.

### C++ / Qt specifics

- Document in the **header**, on the declaration the caller sees.
- Use `///` or `/** */` for API docs, or the plain `/* ... */` field style already used in
  `fortconf.h`. **There is no Doxygen build** — do not add tag ceremony (`@brief`, `@param`) that
  nothing renders and nobody checks. Prose that answers effect/default/edges/units wins.
- **Document what the caller cannot see:** ownership (raw `T*` — who deletes it? Qt parent?),
  nullability, thread affinity (may this be called off the GUI thread?), whether a getter is cheap
  or does I/O, and which values are invalid.
- **Document failure.** C++ has no checked exceptions and this codebase mostly reports failure by
  return value (`bool`, `hasError()` / `errorMessage()` as on `ConfBuffer`). If a function can fail,
  say **how you learn that** and **what state the object is left in**.
- Ranges/limits get their bounds named — e.g. `ConfUtil::ruleMaxCount()` style limits exist because
  the driver has fixed capacity; say so.
- **Kernel code (`src/driver/`) has the strictest duty:** document **IRQL requirements**, locking
  (which lock must be held), allocation pool, and buffer-length preconditions. These are the
  invariants that turn into a BSOD when a caller guesses wrong. They are invisible in the signature.

### Entry points get a worked example

Public API that is the front door of a subsystem (e.g. `ConfBuffer::writeConf`, `Device::ioctl`,
`WorkerManager::enqueueJob`) carries a short usage example, or a one-line statement of the required
call order when order matters. **Anything with a call-order or buffer-state precondition must say
so** — that is knowledge no signature carries.

Real example of why: `ConfBuffer::writeVersion()` looks like a harmless "add the version header"
call. It is not — it **`resize()`s the buffer down to `sizeof(FORT_CONF_VERSION)`**, discarding
whatever was written. It is meant to be used **standalone**, to build the small validate-IOCTL
payload (`confmanager.cpp:881`), *not* as a prologue to `writeConf()`. Nothing in the name or the
signature tells you that, and getting it backwards silently truncates the config blob sent to the
kernel. One line of doc on the declaration prevents that.

Likewise say **how failure is reported**: `writeConf()` returns `bool`, and the reason is only
available via `hasError()` / `errorMessage()`. A caller who checks neither gets a silent wrong
result.

## Inline comments explain WHY, never what

```cpp
// BAD — restates the code
// Multiply kBits by 128
limit->bps = quint64(kBits) * (1024LL / 8);

// GOOD — explains why this constant, sourced
// Driver's FORT_SPEED_LIMIT.bps is bytes/s; UI stores kbits/s. 1024/8 keeps the
// UI's "Kb/s" combo values (10, 20, ... 1024) aligned with the KiB-based labels.
limit->bps = quint64(kBits) * (1024LL / 8);
```

- Comment the **non-obvious**: a magic constant's source, a workaround and the bug it dodges, a
  deliberate deviation from the obvious approach, an ordering constraint, an upstream quirk.
- **No commented-out code.** Git remembers. Delete it.
- **No stale comments.** A comment that contradicts the code is worse than none — it is believed.
- Keep `_clang-format`'s 100-column limit; it applies to comments too.

## File-level comment review on every edit (MANDATORY)

When you edit a **pre-existing** file, review the **whole file's** comments — not just your lines —
and bring it up to this standard **in the same change**. Each touched file leaves compliant.

Checklist for the file you touch:
- Public API you changed states effect + default + edge values + **units**.
- Inline comments explain **why**; delete what-comments, stale comments, commented-out code.
- Kernel files: IRQL/locking/pool/buffer-length preconditions documented.

**Keep it proportional.** This is comment hygiene on a file you were already editing — not a licence
to refactor, and not permission to reformat upstream code. In this fork, restraint is the default:
if the file is genuinely fine, change nothing — but you must have looked.

## Report bugs you notice — never silently fix them

Reading a file closely for comments finds real defects. When it does, **tell the owner. Do not fold
the fix into your comment change, do not fix it quietly, do not skip it.**

For each bug report:
- **Severity:** Blocker / High / Medium / Low
- **Location:** `file:line`
- **Symptom:** what actually goes wrong
- **Impact/scope:** who it affects, how often, and how deep it runs — can it crash, BSOD, corrupt
  the config/DB, or silently **allow traffic that should be blocked**? A firewall's wrong answer is
  silent; weigh it accordingly.

A correct-but-unclear comment is a doc fix. A **wrong** comment hiding a code defect is a **bug** —
report it. Then let the owner decide: fix now, file it, or defer. Do not expand scope on your own.
