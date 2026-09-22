# 14. Build and deployment

This document explains how the HLASM sources in this repository become z/OS
load modules, what the repository itself contributes to that process
(`.gitattributes`, `zapp.yaml`, `LINKPARM/*.ftl`, `TABLE/*.csv`,
`JCL/BIND.JCL`), and which parts of the build can only be executed on z/OS.

Nothing in this repository is a complete build system: there is no JCL or
script that assembles the 24 programs. The repository supplies the
*inputs* that a GenevaERS build pipeline (the `PSRCREPO` column of the
`TABLE` files refers to it as `Performance-Engine`) consumes. Where this
document describes what such a pipeline does with those inputs, that is
inference from the file contents and from standard HLASM/Binder behaviour,
and it is marked as such.

---

## 14.1 Prerequisites (from `README.md`)

`README.md` lists two build products:

* Git
* High Level Assembler Toolkit

and dedicates the rest of the file to installing **Rocket Software's Git for
z/OS USS**. The points that matter for this repository are:

* Rocket Git adds *code-page conversion* to Git so EBCDIC files can be stored
  in an ISO8859-1 repository; the mapping is driven by `.gitattributes`
  (see 14.2).
* At least 550 MB of free space is needed for Git, Gzip, Bash and Perl in a
  directory such as `/var/rocket`.
* A sample `gitenv.sh` sets `GIT_SHELL`, `GIT_EXEC_PATH`, `GIT_TEMPLATE_DIR`,
  `PATH`, `MANPATH`, `PERL5LIB`, `LIBPATH` and the ASCII-support variables
  `_BPXK_AUTOCVT=ON`, `_CEE_RUNOPTS="FILETAG(AUTOCVT,AUTOTAG) POSIX(ON)"`,
  `_TAG_REDIR_ERR/IN/OUT=txt`. The script exists because a Jenkins agent
  runs in a non-login shell where `.profile` is not executed.
* An optional `git config --global core.editor "/bin/vi -W filecodeset=ISO8859-1"`
  keeps commit messages in ISO8859-1.

The README does not describe the assembly or link steps; those come from the
ZAPP and link-parameter files below.

---

## 14.2 Source encoding: `.gitattributes`

```text
# Default to EBCDIC unless otherwise specified
*                zos-working-tree-encoding=ibm-1047 git-encoding=iso8859-1

# freemarker templates must be ascii
*.ftl            zos-working-tree-encoding=iso8859-1 git-encoding=iso8859-1
# The git attributes and ignore files MUST be ASCII
*.gitattributes  zos-working-tree-encoding=iso8859-1 git-encoding=iso8859-1
*.gitignore      zos-working-tree-encoding=iso8859-1 git-encoding=iso8859-1
```

Consequences:

```text
   repository (ISO8859-1)                    z/OS USS working tree
   ──────────────────────                    ─────────────────────
   ASM/*.asm, MAC/*.mac, JCL/*, TABLE/*,     ibm-1047 (EBCDIC) - readable by
   README.md, LICENSE, V10727.xml  ───────►  HLASM, TSO, ISPF, JES
   LINKPARM/*.ftl, .gitattributes,           iso8859-1 (ASCII) - readable by
   .gitignore                      ───────►  the FreeMarker template engine
```

* Every HLASM source and macro is checked out as **IBM-1047** on z/OS. HLASM
  reads EBCDIC, so the `.asm`/`.mac` files must never be tagged ASCII on the
  host. On a non-z/OS clone (like the one used to write these documents) the
  files are plain ISO8859-1/ASCII and can be read with ordinary tools.
* `.ftl` files are FreeMarker templates and stay ASCII on both sides because
  the template engine (part of the build tooling, not this repository) runs in
  the ASCII world of USS/Java.
* Characters that differ between code pages are the usual suspects in HLASM:
  `¬` (NOT), `|` (OR), `[`/`]`, `{`/`}`. The sources use `X'..'` and
  mnemonic forms (`OC`, `NILF`, `XGR`) rather than special characters, which
  is what allows a round trip through ISO8859-1 without loss.

There is no `.gitignore` in the repository even though `.gitattributes`
reserves an encoding for it.

---

## 14.3 HLASM Toolkit / ZAPP configuration: `zapp.yaml`

```yaml
name: sam
description: GenevaERS Z APPlication (ZAPP) file
version: 3.0.0
author:
  name: GenevaERS team

propertyGroups:
  - name: hlasm-local
    language: hlasm
    libraries:
      - name: syslib
        type: local
        locations:
          - "**/MAC"
```

The ZAPP (Z APPlication) file is the descriptor read by IBM's Z Open Editor /
Wazi tooling and the HLASM language server. The single property group tells
the assembler front end that, for `hlasm` sources, the `SYSLIB`
macro/copybook library is the local `MAC` directory. That is all the
repository says about assembly; it is enough because every program pulls its
DSECTs and macros with `COPY`/macro calls that resolve against `MAC`:

```text
ASM/GVBMR95.asm ──COPY/macro──► MAC/GVBMR95W.mac  (work area)
                              MAC/GVBMR95C.mac  (THRDAREA, PARMTBL, ...)
                              MAC/GVBMR95L.mac  (LOGICTBL + LTREDEFN overlays)
                              MAC/GVBMSG.mac    (GVBMSG LOG/WTO/FORMAT)
                              MAC/GVBUTEQU.mac  (message equates)
                              ...
```

`TABLE/MAC.csv` (see 14.7) enumerates the 92 members of `MAC` that the build
treats as HLASM macros/copybooks; the `MAC` directory contains exactly those
92 files.

---

## 14.4 Link-edit templates: `LINKPARM/*.ftl`

Each file in `LINKPARM` is a FreeMarker template whose output is a Binder
(`IEWL`/`IEWBLINK`) control-statement stream for **one load module**. Only
15 templates exist; the other nine programs in `TABLE/PGM.csv` are marked
`OBJONLY` and are bound *into* those 15 modules by the Binder's automatic
library call (they are referenced with `V(...)` constants and have no
template of their own).

```text
LINKPARM template   Load module   Statically bound OBJONLY members (autocall)
─────────────────   ───────────   ──────────────────────────────────────────────────
GVBMR95.ftl         GVBMR95       GVBMR96 (INCLUDE), GVBMRAD (INCLUDE),
  aliases GVBMR95E,               GVBMRSQ + GVBMRSU (INCLUDE when GERS_DB2_ASM=Y),
          GVBMR95R                GVBMRBS, GVBMRVK, GVBSRCHR, GVBUTMUE (via GVBUTMSG)
GVBMR88.ftl         GVBMR88       GVBMR87
GVBMRHPU.ftl        INZEXIT       (entry INZEXIT, AC(1))
GVBDAYS.ftl         GVBDAYS
GVBDL96.ftl         GVBDL96
GVBTP90.ftl         GVBTP90
GVBUR20.ftl         GVBUR20
GVBUR33.ftl         GVBUR33
GVBUR35.ftl         GVBUR35
GVBUR39.ftl         GVBUR39
GVBURALI.ftl        GVBURALI
GVBURZTM.ftl        GVBURZTM
GVBUT99.ftl         GVBUT99
GVBUTHDR.ftl        GVBUTHDR
GVBUTMSG.ftl        GVBUTMSG      GVBUTMUE
```

The "autocall" column is inference: the templates do not `INCLUDE` those
members, but the programs contain strong `V(...)` references to them
(`GVBMRBS DC V(GVBMRBS)`, `GVBMRVK DC V(GVBMRVK)`, `GVBSRCHR DC V(GVBSRCHR)`
in `GVBMR95`, `GVBMR87A DC V(GVBMR87)` in `GVBMR88`,
`L R11,=V(GVBUTMUE)` in `GVBUTMSG`), and the Binder resolves such references
from `SYSLIB` unless `NCAL` is specified. The same mechanism binds copies of
`GVBDL96`, `GVBDAYS`, `GVBTP90`, `GVBUR35`, `GVBUTHDR` and `GVBUTMSG` into
`GVBMR95`/`GVBMR88`, which is why those utilities are `LOADMOD` (callable on
their own) *and* present inside the engines.

### 14.4.1 The engine template (`GVBMR95.ftl`)

```text
 SETOPT  PARM(HOBSET=YES)
 SETOPT  PARM(RMODE=SPLIT)
<#if env["GERS_DB2_ASM"] == "Y">
 INCLUDE SYSLIB(GVBMRSQ)
 INCLUDE SYSLIB(GVBMRSU)
</#if>
 INCLUDE SYSLIB(GVBMRAD)
 INCLUDE SYSLIB(GVBMR95)
 INCLUDE SYSLIB(GVBMR96)
 ENTRY   GVBMR95
 ALIAS   GVBMR95E,GVBMR95R
 NAME    GVBMR95(R)
```

* `HOBSET=YES` - the Binder sets the high-order bit of the entry-point and
  alias addresses according to each entry's AMODE, so a `LOAD`/`LINK` of the
  module enters in AMODE 31 as the `GVBMR95 AMODE 31` statement requests. The
  source comment explains why the entry AMODE is 31 rather than 64: the
  `IDENTIFY` entries created at run time inherit the entry AMODE and some are
  used by external callers, so the program enters in AMODE 31 and issues
  `SAM64` immediately.
* `RMODE=SPLIT` - the module is loaded as two class segments: the RMODE 24
  parts below the line and the RMODE 31/ANY parts above it. `GVBMR95.asm`
  itself declares `GVBMR95 RMODE 31` and a second CSECT `MACHCODE RMODE 31`
  (the model-code segment table), `GVBMR96` is `RMODE ANY`, but the drivers
  `GVBMRBS`, `GVBMRVK` and `GVBMRSQ` are `RMODE 24` because they hold BSAM /
  VSAM control blocks and DB2 CAF parameter lists that must be addressable
  by 24-bit services. SPLIT lets the same load module hold both.
* The `<#if env["GERS_DB2_ASM"] == "Y">` block is the only conditional in
  any template. When the build environment exports `GERS_DB2_ASM=Y`, the DB2
  SQL and DB2 HPU drivers are bound in and the weak `WXTRN GVBMRSQ` /
  `WXTRN GVBMRSU` references in `GVBMR95` resolve; otherwise
  `MRSQADDR`/`MRSUADDR` stay zero and the engine reports
  `DB2_SQL_UNAVAILABLE` (197) or `DB2_HPU_UNAVAILABLE` (198) when a view
  needs them (see [09-io-handlers.md](09-io-handlers.md)). `GVBMRSQ` is also
  the only program whose `PDB2PRE` column in `TABLE/PGM.csv` is `Y`, i.e. it
  needs the DB2 precompiler (`EXEC SQL` statements) before HLASM.
* `GVBMRAD` (Adabas) is always included. It only contains the *engine-side*
  driver; the Adabas `ADABAS`/`ADAUSER` interface itself is site software
  resolved at run time.
* `GVBMRZP` (zIIP/SRB support) and `GVBMRDV`/`GVBMRDI` (DB2-via-VSAM) are
  `WXTRN` in `GVBMR95`/`GVBMR96` and are **not** in this repository or any
  template; a site that has them binds them by adding `INCLUDE` statements.
  Without them the corresponding features report
  `ZIIP_FEATURE_NOT_AVAILABLE` (511) / `IO_DRIVER_UNAVAILABLE` (141).
* `ENTRY GVBMR95` with `ALIAS GVBMR95E,GVBMR95R`: all three names enter the
  same CSECT. `GVBMR95` reads the name it was invoked under (via `GVBURALI`
  and the `namepgm` field) and refuses the base name with `EXEC_PGM_ERR`
  (140); `GVBMR95E` selects the extract phase and `GVBMR95R` the reference
  phase (see [03-gvbmr95-extract-engine.md](03-gvbmr95-extract-engine.md)).
  JCL must therefore use `EXEC PGM=GVBMR95E` or `PGM=GVBMR95R`.
* `NAME GVBMR95(R)` - `(R)` replaces an existing member.

### 14.4.2 Attribute matrix for the other templates

All templates except `GVBMR95.ftl` set four `SETOPT PARM(...)` values
explicitly:

| Template | `AMODE` | `RMODE` | `REUS` | `HOBSET` | Extra | Entry |
|---|---|---|---|---|---|---|
| `GVBMR88.ftl` | 31 | 24 | NONE | YES | | `GVBMR88` |
| `GVBMRHPU.ftl` | 31 | 24 | RENT | YES | `AC(1)` | `INZEXIT` |
| `GVBDAYS.ftl` | 31 | ANY | RENT | NO | | `GVBDAYS` |
| `GVBDL96.ftl` | 31 | 24 | RENT | NO | | `GVBDL96` |
| `GVBTP90.ftl` | 31 | 24 | RENT | NO | | `GVBTP90` |
| `GVBUR20.ftl` | 31 | 24 | RENT | YES | | `GVBUR20` |
| `GVBUR33.ftl` | 31 | 24 | RENT | NO | | `GVBUR33` |
| `GVBUR35.ftl` | 31 | 24 | RENT | YES | | `GVBUR35` |
| `GVBUR39.ftl` | 31 | 24 | NONE | NO | | `GVBUR39` |
| `GVBURALI.ftl` | 31 | ANY | RENT | NO | | `GVBURALI` |
| `GVBURZTM.ftl` | 31 | 24 | NONE | NO | | `GVBURZTM` |
| `GVBUT99.ftl` | 31 | 24 | NONE | NO | | `GVBUT99` |
| `GVBUTHDR.ftl` | 31 | ANY | RENT | NO | | `GVBUTHDR` |
| `GVBUTMSG.ftl` | 31 | ANY | NONE | NO | | `GVBUTMSG` |

Reading the matrix:

* **`REUS=RENT`** marks a module reentrant. The reentrant utilities
  (`GVBDL96`, `GVBTP90`, `GVBUR20`, `GVBUR33`, `GVBUR35`, `GVBDAYS`,
  `GVBURALI`, `GVBUTHDR`) obtain their work areas with `STORAGE OBTAIN` or
  receive them from the caller and never store into their own CSECT; they can
  therefore be shared by the parallel extract threads. `GVBMR88`, `GVBUR39`,
  `GVBURZTM`, `GVBUT99` and `GVBUTMSG` are `REUS=NONE` (not marked
  reusable at all) and are loaded afresh for each use; `GVBMR88` in
  particular stores into its own CSECT (`SNAPDCB`, `SORTRSA`, save areas).
* **`RMODE=24`** keeps a module below the 16 MB line. It is used by every
  module that owns BSAM/QSAM/VSAM control blocks or SVC 99 request blocks
  (`GVBMR88`, `GVBTP90`, `GVBUR20`, `GVBUR35`) and by `GVBDL96`, `GVBUR33`,
  `GVBUR39`, `GVBURZTM` and `GVBUT99`. `RMODE=ANY` is used only by the
  pure-computation modules `GVBDAYS`, `GVBURALI`, `GVBUTHDR` and `GVBUTMSG`.
  (The template values are facts; the reason for each choice is inferred
  from the module contents.)
* **`HOBSET=YES`** appears on the modules whose entry AMODE must be
  guaranteed by the Binder-set high-order bit (`GVBMR88`, `GVBUR20`,
  `GVBUR35`, `GVBMRHPU`), all of which are entered from JCL or from other
  products (`GVBUR20` from `GVBMR96`'s I/O layer, `INZEXIT` from DB2 HPU).
* **`AC(1)`** on `GVBMRHPU.ftl` gives `INZEXIT` an authorization code of 1.
  The module is loaded by the DB2 High Performance Unload utility as an
  initialization exit, and `GVBMRSU` also requires the *job step* to be
  APF-authorized (`TESTAUTH` in `GVBMRSU`, see
  [09-io-handlers.md](09-io-handlers.md)); the load library that receives
  `INZEXIT` must be in the APF list for `AC(1)` to have any effect.
* No `AMODE 64` module exists: `GVBMR95`/`GVBMR96` run their main loops in
  AMODE 64 (`SYSSTATE ARCHLVL=2,AMODE64=YES`, `SAM64`) but enter in AMODE 31
  as described above, and switch back with `SAM31` before every call to a
  31-bit driver or utility (`BASSM` transitions in
  [09-io-handlers.md](09-io-handlers.md)).

### 14.4.3 Assemble-and-link pipeline (inferred)

The repository does not contain the JCL or shell scripts that drive the
build. Combining the README, `zapp.yaml`, `TABLE/PGM.csv` and the templates,
the pipeline must look like this:

```text
   TABLE/PGM.csv          TABLE/MAC.csv
   (24 programs)          (92 macros)
        │                      │
        ▼                      ▼
  ┌──────────────┐  SYSLIB  ┌──────────┐
  │ ASM/*.asm    │◄─────────│ MAC/*.mac│
  └──────┬───────┘          └──────────┘
         │  GVBMRSQ only (PDB2PRE=Y): DB2 precompiler  ──► DBRM  ──► JCL/BIND.JCL
         ▼
  ┌──────────────┐
  │  HLASM       │  ASMA90, SYSLIB=MAC
  └──────┬───────┘
         ▼  object decks (OBJ library = "SYSLIB" in the templates)
  ┌──────────────┐
  │ FreeMarker   │  LINKPARM/<pgm>.ftl + env (GERS_DB2_ASM) ──► binder control cards
  └──────┬───────┘
         ▼
  ┌──────────────┐
  │  Binder      │  IEWL: SETOPT/INCLUDE/ENTRY/ALIAS/NAME, autocall from object library
  └──────┬───────┘
         ▼
     LOADLIB: GVBMR95 (+GVBMR95E/GVBMR95R), GVBMR88, GVBDL96, GVBTP90, ... INZEXIT
```

Points the sources impose on any such pipeline:

* `GVBMR95` and `GVBMR96` must be assembled with the same `MAC` versions;
  `GVBMR96` builds structures (`LOGICTBL`, `THRDAREA`, literal pools) that
  `GVBMR95` and its generated code address by offset, and the macros contain
  `ASSERT`s (`ASSERT stdparml,eq,execdlen`,
  `ASSERT (ltwr_next_exit-logictbl),EQ,(ltre_next_exit-logictbl)`) that make
  HLASM fail the assembly if the layouts drift.
* `GVBSRCHR` is generated code: its source expands a macro loop that emits
  one binary-search routine per key length 1-256. It is large but has no
  external dependencies beyond `MAC`.
* `GVBUTMUE` is the object form of the message catalogue: its whole source
  is `GVBUTMUE GVBMSGGE CASE=Mixed`, and `MAC/GVBMSGGE.mac` expands every
  `GVBMSGDF` definition into the directory and text tables. A message-text
  change requires re-assembling `GVBUTMUE` **and** re-linking `GVBUTMSG`,
  `GVBMR95` and `GVBMR88` (all of which statically include `GVBUTMUE` via
  `GVBUTMSG`).
* `GVBMR95.asm` is ~28,000 lines and holds two CSECTs (`GVBMR95` and
  `MACHCODE`); HLASM needs a correspondingly large `SYSUT1`/region.

---

## 14.5 Db2 binding: `JCL/BIND.JCL`

`JCL/BIND.JCL` is the only JCL in the repository. It binds the DBRM produced
by pre-compiling `GVBMRSQ` into a package and a plan, and grants execute
authority. It applies only when `GERS_DB2_ASM=Y` was used for the link.

```jcl
//         EXPORT SYMLIST=*
//         SET HLA=DSN
//         SET HLB=V13R1M0
//         SET HL1=&HLA..&HLB.
//         SET SUBSID=DM13
//         SET DSNAME='GEBT.LATEST.GVBDBRM'
//         SET LOCNAME=DM13
//         SET COLLID=ERS01
//         SET SPLANSFX=X
//         SET PLAN=GVBMRSQ&SPLANSFX
//         SET MEMBER=GVBMRSQ
//         SET SCHEMA=SAFRWBGD
//         SET OWNER=USERID1
//         SET SYSADM=USERID2
//         SET RUNLIB='DSN131.RUNLIB.LOAD'
//         SET TIAD=DSNTIA13
//PACKAGE EXEC PGM=IKJEFT01,DYNAMNBR=20,COND=(4,LT)
//STEPLIB DD  DISP=SHR,DSN=&HL1..SDSNEXIT
//        DD  DISP=SHR,DSN=&HL1..SDSNLOAD
//DBRMLIB DD  DISP=SHR,DSN=&DSNAME.
//SYSTSIN DD *,SYMBOLS=EXECSYS
  DSN SYSTEM(&SUBSID.)
  BIND PACKAGE(&COLLID.) OWNER(&SYSADM.) MEMBER(&MEMBER.) CURRENTDATA(YES) -
       ENCODING(EBCDIC) ACTION(REPLACE) RELEASE(COMMIT) QUALIFIER(&SCHEMA) -
       VALIDATE(BIND) EXPLAIN(YES) ISOLATION(CS) LIB('&DSNAME.') FLAG(I)
  BIND PLAN(&PLAN.) PKLIST(&COLLID..*) ACTION(REPLACE) ISO(CS) -
       CURRENTDATA(YES) QUALIFIER(&SCHEMA) OWNER(&SYSADM.) ENCODING(EBCDIC)
  RUN PROGRAM(DSNTIAD) PLAN(&TIAD.) LIB('&RUNLIB.')
//SYSIN   DD *,SYMBOLS=EXECSYS
SET CURRENT SQLID='&SYSADM.';
GRANT EXECUTE ON PLAN &PLAN. TO PUBLIC;
```

(The `BIND` statements above are condensed onto fewer lines; the JCL uses
one option per line with `-` continuation.)

How this connects to the runtime:

```text
 JCL/BIND.JCL                              GVBMR96 / GVBMRSQ at run time
 ────────────                              ────────────────────────────
 SET PLAN=GVBMRSQ&SPLANSFX  (GVBMRSQX) ◄── DB2_SQL_PLAN_NAME=... in EXTRPARM
                                            (EXECSPLN, default 'GVBMRSQ')
 SET MEMBER=GVBMRSQ  (package = DBRM) ◄──── EXEC SQL statements in ASM/GVBMRSQ.asm
 SET SCHEMA=SAFRWBGD (QUALIFIER)      ◄──── unqualified table names in view SQL
 DSN SYSTEM(DM13)                     ◄──── DSNALI CONNECT subsystem chosen by GVBMRSQ
```

* `PLAN` defaults to `GVBMRSQ` with a site suffix (`SPLANSFX=X` gives
  `GVBMRSQX`). Whatever name is bound must be supplied to the engine through
  `DB2_SQL_PLAN_NAME` (default `GVBMRSQ`, see
  [13-parameters-ddnames-messages.md](13-parameters-ddnames-messages.md)),
  because `GVBMRSQ` opens the plan through the Call Attach Facility
  (`DSNALI` `OPEN`) with the name in `EXECSPLN`.
* `ISOLATION(CS)`/`CURRENTDATA(YES)` and `RELEASE(COMMIT)` are the bind-time
  choices for the read-only cursors that `GVBMRSQ` prepares dynamically
  (`PREPARE`/`OPEN`/`FETCH`); the SQL text itself comes from the VDP at run
  time, so `VALIDATE(BIND)` only checks the static statements of the driver.
* `EXPLAIN(YES)` requires `PLAN_TABLE` under `OWNER`.
* `DSNTIAD` (dynamic SQL processor) executes the `GRANT` so that any job can
  run the plan.
* The other two plan names known to the engine, `GVBMRDV` (`DB2_VSAM_PLAN_NAME`)
  and `GVBMRCT` (`DB2_CATALOG_PLAN_NAME`), belong to modules that are not in
  this repository; no bind JCL is provided for them.

---

## 14.6 Runtime deployment

Once the load library exists the runtime footprint is:

```text
 STEPLIB / JOBLIB
 ├─ GVBMR95   (GVBMR95E, GVBMR95R)   extract + reference phases
 ├─ GVBMR88                          format phase (SORT E35 or stand-alone)
 ├─ GVBDL96, GVBDAYS, GVBTP90, GVBUR20, GVBUR33, GVBUR35, GVBUR39,
 │  GVBURALI, GVBURZTM, GVBUTHDR, GVBUTMSG      shared utilities
 ├─ GVBUT99                          abend-step utility
 └─ INZEXIT                          DB2 HPU exit (APF library, AC(1))
 site-supplied, optional:
 ├─ GVBMRZP                          zIIP/SRB support (WXTRN)
 ├─ GVBMRDV, GVBMRDI                 DB2-via-VSAM drivers (WXTRN)
 ├─ ADABAS / ADAUSER                 Adabas interface used by GVBMRAD
 └─ DFSORT or SyncSort               invoked by GVBMR88 and between phases
```

Deployment constraints that follow directly from the source:

| Requirement | Why |
|---|---|
| `EXEC PGM=GVBMR95E` / `GVBMR95R`, never `GVBMR95` | `EXEC_PGM_ERR` (140) - alias enforcement in `GVBMR95` |
| APF-authorized STEPLIB for `PAGE_FIX_IO_BUFFERS=Y`, `USE_ZIIP=Y`, DB2 HPU | `TESTAUTH FCTN=1` in `GVBMR96`/`GVBMRSU`; `MODESET KEY=ZERO,MODE=SUP` in the zIIP path |
| `REGION=0M` or a large region | reference pools are 64-bit (`IARV64`) but thread areas, logic tables and I/O buffers are 31-bit `STORAGE OBTAIN`; `IO_BUFFER_LEVEL` multiplies BSAM buffers per thread |
| Same `MAC` level across all modules in the library | offset-based access between `GVBMR96`, `GVBMR95`, generated code and `GVBMR88`; `VERIFY_CREATION_TIMESTAMP` only checks VDP vs. logic table, not module levels |
| Db2 plan bound with `JCL/BIND.JCL` when DB2 views are used | `DSNALI OPEN` with `EXECSPLN` |
| `GVBURALI` present | `GVBMR95` calls it at entry to discover the alias it was invoked under (`namepgm`) |

---

## 14.7 Build metadata: `TABLE/*.csv`

The `TABLE` directory holds the tables a GenevaERS build database imports to
know what to assemble.

`TABLE/tablesPE.csv` lists the tables:

```text
name
PGM
MAC
```

`TABLE/PGM.csv` (columns `PID,PDESC,PPGMTYPE,PSRCREPO,PSRCFLDR,PSRCSFX,PLANG,PMODTYPE,PDB2PRE`):

| `PID` | `PPGMTYPE` | `PMODTYPE` | `PDB2PRE` | Meaning for the build |
|---|---|---|---|---|
| `GVBMR95`, `GVBMR88`, `GVBUT99`, `GVBMRHPU` | Main Program | LOADMOD | | executed from JCL (or by HPU) |
| `GVBDAYS`, `GVBDL96`, `GVBTP90`, `GVBUR20`, `GVBUR33`, `GVBUR35`, `GVBUR39`, `GVBURALI`, `GVBUTHDR`, `GVBUTMSG` | Subprogram | LOADMOD | | callable load modules with their own `.ftl` |
| `GVBURZTM` | Subroutine | LOADMOD | | has an `.ftl` although classified as subroutine |
| `GVBMR87`, `GVBMR96`, `GVBMRBS`, `GVBMRAD`, `GVBMRSU`, `GVBMRVK`, `GVBSRCHR`, `GVBUTMUE` | Subroutine | OBJONLY | | object only; bound into another module |
| `GVBMRSQ` | Subroutine | OBJONLY | `Y` | needs the DB2 precompiler first |

All 24 rows have `PSRCREPO=Performance-Engine`, `PSRCFLDR=ASM`,
`PSRCSFX=asm`, `PLANG=HLASM`; the `ASM` directory contains exactly these 24
files.

`TABLE/MAC.csv` (columns `CID,CSRCREPO,CSRCFLDR,CSRCSFX,CLANG`) has 92 rows,
all `Performance-Engine,MAC,mac,HLASM`, one per file in `MAC`. The build uses
it to know which members to copy into the `SYSLIB` PDS(E) before assembling.

The `LOADMOD`/`OBJONLY` classification is what decides whether a `.ftl`
exists: every `LOADMOD` row has a template and no `OBJONLY` row does.

---

## 14.8 The sample view: `V10727.xml`

`V10727.xml` is not a build input. It is a GenevaERS Workbench export
(`<PROGRAM>Workbench Version 4.50.0</PROGRAM>`, `XMLVERSION 3`) of view
`10727` `DEMO_CUSTNAME_ADA` created 2024-01-10. It documents the *shape* of
what the Workbench feeds into the compiler that produces the VDP and logic
table consumed by `GVBMR95R`/`GVBMR95E`; the engine itself never reads XML.
The `_ADA` suffix and the Adabas-related fields make it a companion to the
`GVBMRAD` driver and to the `CALLADA` access method (17).

---

## 14.9 What can be validated off-host versus on z/OS

```text
                       Linux / macOS clone              z/OS USS + HLASM Toolkit
                       ───────────────────              ────────────────────────
 read sources                 yes (ASCII)                       yes (IBM-1047)
 macro/DSECT cross-checks     yes (text tools)                  yes
 HLASM syntax check           no  (no ASMA90)                   yes (Z Open Editor / batch)
 assembly, ASSERTs            no                                yes
 FreeMarker → binder cards    only with a JVM + FreeMarker      yes (build tooling)
 Binder / load modules        no                                yes
 DB2 precompile + BIND        no                                yes (JCL/BIND.JCL)
 execution, SNAP, TRACE       no                                yes
```

Nothing in this repository can be assembled or executed off-host: HLASM,
the Binder, DFSORT, DB2 and the z/OS services (`STORAGE`, `ATTACHX`,
`IEA4xxx`, `IWM4xxx`, `ESTAEX`, `IARV64`) exist only on z/OS. These
documents were produced by reading the source; every statement about
run-time behaviour is derived from the code paths described in documents
03-13 and has not been confirmed by executing the engine.

---

## 14.10 Related documents

* [01-overview.md](01-overview.md) - repository layout and program inventory.
* [02-architecture.md](02-architecture.md) - addressing modes and inter-module linkage.
* [09-io-handlers.md](09-io-handlers.md) - the drivers whose optional inclusion the `GVBMR95.ftl` template controls.
* [13-parameters-ddnames-messages.md](13-parameters-ddnames-messages.md) - the `DB2_*_PLAN_NAME`, `USE_ZIIP` and `PAGE_FIX_IO_BUFFERS` parameters that depend on how the modules were linked and authorized.
