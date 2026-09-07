# AGENTS.md

This file provides guidance to AI coding agents (Claude Code, Codex, and others) working in this repository.
It mirrors `CLAUDE.md`; keep the two in sync.

## What this repository is

CardCore is lowtouch-ai's fork of **AWS CardDemo** — a z/OS mainframe application (COBOL / CICS / VSAM / JCL,
plus optional Db2, IMS DB, and MQ) published by AWS as a test bed for mainframe discovery, migration, and
modernization tooling. This repo holds **source only** — nothing compiles or runs locally. Programs are
compiled and executed on a mainframe (or an emulator such as AWS Mainframe Modernization / Micro Focus /
UniKix) after the source is uploaded into z/OS PDS members.

Because the codebase intentionally exercises analysis tooling, it mixes coding styles and paradigms on purpose.
Do not "normalize" style across programs.

Source artifacts keep the upstream `CardDemo` naming — program banners, `AWS.M2.CARDDEMO.*` dataset names,
`GROUP(CARDDEMO)` in the CSD, and the `Ver: CardDemo_*` stamps. Only prose (README.md, CLAUDE.md, this file,
`docs/`) says "CardCore". Do not rename inside source.

## Current purpose: COBOL-to-Java migration demo

The repo is the demo codebase for a COBOL-to-Java port driven by Krithi, an agentic SDLC engine. Stage 1 of
that pipeline has the REQ agent read source and JCL and produce a PRD — see README.md for the pipeline.

Functional documentation of the application (transaction inventory, screen list, program-to-function mapping,
business rules, batch cycle, data model) lives in `docs/` — see `docs/README.md` for the index. The root
README.md stays free of business detail by convention (it describes the migration pipeline, not the
application); put functional documentation in `docs/` instead. The three optional-module READMEs under
`app/app-*/` and the screen/flow images in `diagrams/` also carry functional detail specific to those modules.

**The modern target — Java (Spring Boot) API + Next.js UI — lives in two sibling repos, not here.** This
repo (`cardcore`) is the legacy COBOL source only, read-only input to the REQ agent; nothing in it is ever
ported in place. The migration output goes to:

- **`cardcore-api`** — Spring Boot 3 / Java 17 API. Entities mirror the legacy VSAM record layouts
  (`Customer` ~ `CVCUS01Y`, `Account` ~ `CVACT01Y`, `Card` ~ `CVACT02Y`, `CardXref` ~ `CVACT03Y`,
  `Transaction` ~ `CVTRA05Y`) — see `docs/data-model.md` in this repo for the source layouts they're
  ported from.
- **`cardcore-ui`** — Next.js 14 (App Router), TypeScript, Tailwind. Calls `cardcore-api` server-side
  (never a client-side base URL) for the screens it renders.

An agent (or reviewer) working from PRD/FRS output produced against *this* repo should look in those two
sibling repos for the actual Java/Next.js implementation state — this repo's own `app/` tree never gains a
Java or TypeScript equivalent; the port lands there instead.

**Hard rule for agents: this repo (`cardcore`) is read-only source for the migration.** Never edit, add, or
delete anything under `app/` (COBOL, copybooks, BMS, JCL, CSD, etc.) as part of a migration task — the COBOL
application is the *input* being analyzed, not something being ported in place. All migration output goes to
the sibling repos, split strictly by layer:

- Any UI/screen/frontend change → `cardcore-ui` only.
- Any API/business-logic/data-model change → `cardcore-api` only.

If a migration task seems to require touching COBOL source in this repo, that's a signal the task is scoped
wrong — stop and flag it instead of editing `app/`. Editing files in `docs/`, `README.md`, `CLAUDE.md`, or
`AGENTS.md` (documentation/prose about the migration) is not covered by this rule.

## Build and run

There is no local build. Compilation and execution are JCL jobs on the target system:

| Artifact type | Sample JCL | Proc invoked |
|:--|:--|:--|
| BMS mapset (compile **before** its program) | `samples/jcl/BMSCMP.jcl` | `samples/proc/BUILDBMS.prc` |
| CICS COBOL program | `samples/jcl/CICCMP.jcl` | `samples/proc/BUILDONL.prc` |
| Batch COBOL program | `samples/jcl/BATCMP.jcl` | `samples/proc/BUILDBAT.prc` |
| CICS + Db2 program (adds DSNHPC precompile) | `samples/jcl/CICDBCMP.jcl` | `samples/proc/BLDCIDB2.prc` |
| CICS + IMS + MQ program | `samples/jcl/IMSMQCMP.jcl` | inline steps (DFHECP1$ → IGYCRCTL → IEWL) |

Each sample JCL has `SET MEMNAME=` / `SET HLQ=` at the top — substitute the member name and HLQ. Runtime bundles
for emulators live in `samples/m2/`.

The dev loop is driven by `scripts/*.sh`, all of which push files over an **FTP tunnel on `localhost:2121`**
(`tnftp`) to a remote mainframe. They exit early with "FTP Tunnel to Ensono not running." if no tunnel exists,
so they are inert without one. Run them from the repo root (paths inside are relative, e.g. `jcl/<JOB>.jcl`).

- `scripts/upld_module.sh <path> <TYPE>` — pads records to 80 cols via `scripts/pad.awk` (**this rewrites the
  source file in place**) then FTPs it to `AWS.M2.CARDDEMO.<TYPE>(<MEMBER>)`.
- `scripts/remote_compile.sh <file> <.cbl> <basename>` — substitutes `ZZZZZZZZ` in
  `scripts/compile_batch.jcl.template` and submits it. Note it calls `make -f Makefile`, and **no Makefile exists
  in the repo** — that line will fail; upload modules with `upld_module.sh` first.
- `scripts/remote_submit.sh <file> <.jcl>` — submits a single job.
- `scripts/remote_refresh.sh` — reload all VSAM files from sample data (CLOSEFIL → loads → OPENFIL).
- `scripts/run_full_batch.sh`, `run_posting.sh`, `run_interest_calc.sh` — the batch cycles from README's
  "Running Batch Jobs" table.
- `scripts/git-addSrcVersionInfo.sh <file>` — stamps/refreshes the `Ver: CardDemo_<git-describe> Date: ...`
  comment footer found at the bottom of most sources. It **deletes all git tags** except `app_version` (hardcoded
  `v2.0` in the script) before computing the version — don't run it casually on a repo whose tags matter.

There is no test suite. Verification means running the batch jobs / CICS transactions on a live system.

## Layout

`app/` is the base application, one directory per z/OS member type — each maps 1:1 to a PDS that must be
allocated on the target system:

`cbl/` COBOL · `cpy/` copybooks · `bms/` BMS map source · `cpy-bms/` **generated** symbolic maps ·
`jcl/` jobs · `proc/` procs · `asm/` + `maclib/` Assembler utilities · `csd/` CICS resource definitions
(`CARDDEMO.CSD`, applied with DFHCSDUP) · `ctl/` control cards · `catlg/` LISTCAT output ·
`scheduler/` CA7 and Control-M job-stream definitions · `data/` sample data in both `EBCDIC/`
(upload **binary**, dataset-named files) and `ASCII/` (readable equivalents).

Optional modules live in `app/app-authorization-ims-db2-mq/`, `app/app-transaction-type-db2/`, and
`app/app-vsam-mq/`. Each **mirrors the same directory layout** (plus `ddl/`, `dcl/`, `ims/`) and has its own
README. Base application code must keep compiling and running when these are not installed.

`docs/` holds functional documentation of the base application — screens/transactions, batch jobs, data model,
business rules. See `docs/README.md` for the index. Keep it in sync with `app/cbl/`, `app/bms/`, `app/cpy/`,
and `app/jcl/` when those change.

## Mechanics that bite

**Symbolic maps are generated, not hand-written.** `app/cpy-bms/*.CPY` is BMS assembler output for
`app/bms/*.bms`. Edit the `.bms` source, recompile the mapset, and refresh the checked-in `.CPY` — never edit
only one side, or the generated field names drift out of sync with the program that copies them.

**Batch is file-driven.** `SELECT ... ASSIGN TO DALYTRAN` in the COBOL binds to the `//DALYTRAN DD` in the
JCL. Adding or renaming a file means editing both the program's FILE-CONTROL and every JCL that runs it.

**Copybooks are shared contracts.** A copybook under `app/cpy/` is copied into many programs; changing its
layout requires recompiling every program that copies it. Check with grep before editing one.

**Datasets are hardcoded to HLQ `AWS.M2`** throughout `app/jcl/`, `app/csd/CARDDEMO.CSD`, and the scripts. A
different HLQ means a global change across all three. CICS file names in the CSD are matched by 8-character
blank-padded literals in the online programs (`PIC X(08) VALUE 'USRSEC  '`) — mind the trailing blanks.

## Editing conventions

- **Fixed-format COBOL, 80 columns.** Code in cols 8–72, `*` in col 7 for comments, sequence area unused
  (compiles use `NOSEQ`/`NUMBER`). Never let a line exceed col 72 — it is silently truncated by the compiler and
  by the 80-col padding on upload.
- JCL/PROC comments are `//*`, BMS comments are `*` in col 1.
- Every source starts with the Apache 2.0 header block and a `Program/Application/Type/Function` banner; most end
  with the `Ver: CardDemo_...` stamp. Preserve both when editing.
- Files carry mixed line endings and mixed-case extensions (`.cbl` vs `.CBL`, `.jcl` vs `.JCL`) from years of
  mainframe round-trips. Match what a file already uses rather than converting it.
- When adding a screen: BMS map + generated symbolic copybook + program + CSD `DEFINE PROGRAM/MAPSET/
  TRANSACTION` entries, and add it to `docs/screens-and-transactions.md`. When adding a batch job: program +
  JCL, and add it to `docs/batch-jobs.md`. Keep functional documentation in `docs/`, not the root README.
