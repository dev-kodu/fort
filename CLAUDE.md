# fort — Project Rules for Claude

## What this repo is

**Fort Firewall** — a standalone Windows firewall (Windows 7+) with its own kernel-mode WFP
driver. C++/Qt user interface + a C kernel driver. Version **3.20.0**, `DRIVER_VERSION 58`
(`src/version/fort_version.h`).

Evidence: `README.md`, `src/FortFirewall.pro`, `src/driver/fortdrv.c`, `src/version/fort_version.h`.
What the code *is* is not in doubt.

### This is a fork, and it is DORMANT

- `origin` = `github.com/dev-kodu/fort`, `upstream` = `github.com/tnodir/fort` (upstream is the
  real project, by Nodir Temirkhodjaev). **Licensed GPL v3** (`LICENSE`) — if we ever distribute a
  modified build, the modified source must be published under GPL v3. Keep that in mind before
  treating this as a private playground.
- **We have contributed nothing**: `git rev-list --count upstream/master..origin/master` = `0`.
  Every commit here is the upstream author's.
- **We are behind**: as of **2026-07-17**, `upstream/master` is **6 commits ahead**
  (tip `95bb760d`, 2026-07-03). **`git fetch upstream` before believing any of these numbers** —
  they were stale by 6 commits when I first read them, and they will be stale again when you read
  this.
- **Our snapshot's last commit is 2026-04-13** (`e20f0b8b`, "UI: Update SQLite to 3.53.0") — the
  upstream author's, not ours. **This repo is DORMANT and stale.** Treat nothing here as current.
- **Why dev-kodu forked it is NOT recorded anywhere in this repo, and I could not determine it.**
  Do not invent a reason. There is no README note, no issue, no branch, no commit of ours.
  The only trace of local intent is an **uncommitted working-tree edit** to
  `ConfUtil::matchWildcard` (see Traps). If you learn the actual purpose, write it here.

### Relationship to other KADE repos: none found

No file references KADE, Tekla, or any sibling repo (`grep -ri "kade\|tekla"` → 0 hits outside
`.git/`). It shares no code, no build, and no dependency with `KADE`, `KADE.Tekla`, `Kade.Scene3D`,
or any other `D:\Dev` repo. Do not wire it to them without an explicit decision recorded here.

## Fork discipline

**`CLAUDE.md` + `docs/` are our only intended divergence from upstream** — agent instructions, no
code. They are additive files upstream does not have, so they do not conflict on rebase. Keep it
that way.

Because we are otherwise at zero divergence, **the cheapest state to be in is zero divergence**.
Before adding anything of our own:

- Prefer contributing upstream, or keep our delta small, isolated, and documented here.
- Never reformat or restructure upstream files for taste — every touched line is a future merge
  conflict against `upstream/master`.
- Rebase on `upstream/master` rather than merging, to keep our delta legible as a patch set.

## Module boundaries

| Path | What it is |
|---|---|
| `src/ui/` | Qt C++ app, built as static lib `FortFirewallUILib` (`FortFirewallUI.pro`) |
| `src/ui_bin/` | The `FortFirewall.exe` entry point; depends on `ui` |
| `src/driver/` | **Kernel-mode WFP driver** (C, WDM). Own MSBuild `fortdrv.vcxproj` |
| `src/driver/loader/` | `fortdl` — driver loader / image mapper (signature-checked, `fort.rsa.pub`) |
| `src/driver/evt/` | Driver event messages (`fortevt.mc` → Windows Event Log resource) |
| `src/driver_payload/` | Driver payload packaging |
| `src/tests/` | GoogleTest/GoogleMock suites (`Common`, `UtilTest`, `StatTest`, `LogBufferTest`, `LogReaderTest`) |
| `src/3rdparty/` | Vendored: `sqlite`, `qcustomplot`, `tlsf`, `tommyds`, `wildmatch`, `breakpad` |
| `src/scripts/`, `deploy/` | i18n / driver scripts; Inno Setup installer + deployment |

**`src/ui/util/conf/` is the load-bearing seam:** it serializes the firewall configuration into the
packed binary blob the kernel driver parses. It is shared ground between C++ and C. See Traps.

Vendored `src/3rdparty/` and `src/ui/3rdparty/` are **upstream code — do not edit**; update them the
way upstream does (e.g. "UI: Update SQLite to 3.53.0").

## Build

> **READ THIS FIRST: our checked-out `master` (`e20f0b8b`) DOES NOT COMPILE.**
> `src/ui/util/conf/confutil.cpp` fails with
> `error C2679: binary '||': no operator found which takes a right-hand operand of type
> 'QRegularExpressionMatch'`. This is an **upstream** defect, fixed upstream *after* our snapshot.
> **Fix: `git merge upstream/master` (or rebase onto it)** — upstream's `e3369bb0` removes the
> broken function. See Traps. Do not hand-patch it; upstream already solved it.

The commands below are **verified on this machine, 2026-07-17**: qmake generates cleanly and the
tree compiles up to the broken file.

Toolchain actually present here:
- Qt **6.9.1** — `D:\Qt\Qt6.9.1\6.9.1\msvc2022_64`
- MSVC **14.51** (Visual Studio **2026** Community, `C:\Program Files\Microsoft Visual Studio\18\Community`)
- Windows SDK / WDK **10.0.26100.0**

```bat
call "C:\Program Files\Microsoft Visual Studio\18\Community\VC\Auxiliary\Build\vcvars64.bat"
md build-win10 & cd build-win10
D:\Qt\Qt6.9.1\6.9.1\msvc2022_64\bin\qmake.exe -o Makefile ..\src\FortFirewall.pro "CONFIG+=release" "CONFIG-=debug"
nmake
```

Output: `build-win10\ui_bin\FortFirewall.exe`. qmake shadow-builds — build from a `build*/` dir;
`.gitignore` ignores `build*/`, so build output never dirties the tree.

**Toolchain drift is real:** the Qt build is `msvc2022_64`, but the MSVC that originally built this
tree (`14.44`, VS 2022 Build Tools, recorded in `build-win10/.qmake.stash`) **is no longer installed**.
MSVC 14.51 compiles everything up to the broken `confutil.cpp` without complaint, so the drift looks
benign — but it is unproven past that point. If you hit ABI/STL oddities, suspect this first.

### Driver — NOT verified here

Needs the WDK + `MSBuild` (neither is on `PATH` in this shell). Upstream's script:

```bat
cd src\driver
msvcbuild.bat x64 win10 x86      REM args: PLAT[x64|Win32] CONFIG[win7|win10] ARCH[x86|arm64]
```
Driver targets are gated behind `CONFIG+=tests` in `src/FortFirewall.pro`, or built directly via
`fortdrv.vcxproj`. Output: `build-driver-win10\<PLAT>\`.

### Tests — NOT verified here, and currently NOT runnable

```bat
qmake ... "CONFIG+=tests"    REM adds driver + tests subdirs
```
Tests need **GoogleTest sources** via the `GOOGLETEST_DIR` env var
(`src/tests/Common/GoogleTest-include.pri`). On this machine `GOOGLETEST_DIR` is **unset** and no
googletest checkout exists, so the suites have never been built here and I could not verify a test
command. `qmake` degrades quietly (`requires(exists(...))` → the tests just drop out of the build) —
**a green build does not mean the tests ran.**

## Style / analyzer reality — know what does NOT exist

- **`src/_clang-format` is the only style config.** WebKit-based Qt style, **100-column limit**,
  Allman braces after functions/classes/structs only (attached elsewhere), pointer binds to the
  name (`char *p`), 4-space indent, sorted includes.
- **There is no `.editorconfig`, no `.clang-tidy`, no analyzer config, and no CI.**
  `.github/` contains only `FUNDING.yml`. Nothing enforces style or warnings on a PR.
- **There is no `TreatWarningsAsErrors` equivalent.** Nothing will catch you. The compiler is the
  only gate, and it is a lenient one.
- **"No CI" is not a theoretical risk here — it already cost 5 months.** A plain type error in
  `ConfUtil::matchWildcard` (see Traps) survived on upstream `master` from 2025-12-05 to 2026-05-10
  because nothing ever compiled `master`. **You are the CI.** Build before you push, every time.
- SonarCloud / PVS-Studio / CodeScene badges in `README.md` are **upstream's** infrastructure and
  do not run for this fork.
- **The codebase has zero doc comments** — of the **283** first-party headers under `src/ui/`
  (excluding vendored `3rdparty/`), **0** contain `///` or `/**`. There is no Doxygen setup. This is
  the existing baseline, not a target: see the documentation standard for
  what that means for *new* code (short version: document what you write; do not retrofit 998 files).

## Traps

- **The dirty working tree is LOAD-BEARING — do not "clean it up".**
  `D:\Dev\fort` has an **uncommitted local modification** to `src/ui/util/conf/confutil.cpp`.
  It is not a stray experiment: **it is the only reason the tree builds.** Reverting it breaks the
  build. The full story, because it explains several things at once:
  - Upstream `9a2f7e35` (2025-12-05) changed `ConfUtil::matchWildcard` — which returns
    `QRegularExpressionMatch` — to `return path.startsWith('[') || StringUtil::match(...);`.
    That is `bool || QRegularExpressionMatch`, and `QRegularExpressionMatch` has no `operator bool`.
    **It does not compile.**
  - It sat on upstream `master` for **13 commits / ~5 months**, because **there is no CI**
    (see below). No release shipped broken: the last tag is `v3.19.9` (2025-10-11) and master has
    been on an unreleased `3.20.0` the whole time.
  - **Our fork snapshot (2026-04-13) landed inside that broken window**, so the tree we have is
    unbuildable as committed. Someone locally patched `confutil.cpp` to make it build and never
    committed it.
  - Upstream fixed it properly in **`e3369bb0`** (2026-05-10, "UI: ConfUtil: Refactor
    matchWildcard()") by **deleting** the function — `matchWildcard` → `wildcardPos()` returning
    `int`, no regex.
  - **The right move is to merge `upstream/master` and drop the local edit**, not to commit our
    variant of a fix upstream has already superseded. Confirm with the owner before discarding it.
- **The UI↔driver binary contract is unforgiving.** `ConfBuffer` (C++) writes a packed blob;
  `fort_device_control_validate` (C, `src/driver/fortdev.c`) rejects it unless
  `conf_ver->driver_version == DRIVER_VERSION`. Any struct/offset change must be made on **both**
  sides and must bump `DRIVER_VERSION` in `src/version/fort_version.h`. A mismatch that slips the
  version gate is parsed by kernel code — i.e. a bugcheck, not an exception.
- **This is a firewall: a wrong result is a security failure, not a cosmetic bug.** A mis-parsed app
  path or a mis-built rule blob silently *allows traffic that should be blocked*. Nothing crashes and
  nothing logs an error. Wildcard/path matching (`ConfUtil`, `ruletextparser`, `3rdparty/wildmatch`)
  is exactly this kind of code — that is why it has the densest tests (`tst_wildmatch.h`,
  `tst_confutil.h`, `tst_ruletextparser.h`). Change it only with tests, and prefer upstream's.
- **Kernel code cannot be debugged the way user code can.** `src/driver/` runs at raised IRQL in WFP
  callouts. There is no exception handling to save you; a bad pointer is a BSOD on the user's
  machine. Prevent, do not catch. See the logging standard.
- The app runs as a **Windows service + a UI process** that talk over RPC (`src/ui/rpc/`,
  `src/ui/control/`). "It works when I run the exe" does not mean it works as a service.
- SQLite databases carry **migrations** (`src/ui/*/migrations/*.sql`) against real user data
  (traffic history, app info). A bad migration destroys data that has no other copy.

## MANDATORY standards

Both apply to **all** code you write or edit in this repo, and both are adapted to *this* repo — do
not import rules from other KADE repos:

- **`docs/CODE-COMMENTS-AND-DOCS.md`** — doc comments on new/changed public API; inline comments
  explain **why**; whole-file comment review on every pre-existing file you edit; **report bugs you
  notice, never silently fix them**.
- **`docs/LOGGING-AND-DIAGNOSTICS.md`** — verbose logging on the risky paths (driver, conf blob,
  IOCTL, DB/migrations, workers, network); log **before** the step that can kill you, per item, and
  flush; timing per phase + `DONE ... in Nms`; logging is best-effort and must never break the
  workflow.

Read the relevant one **before** writing code, not after.
