# 1. Overview

## 1.1 What the Performance Engine is

GenevaERS ("Geneva") is an open-source (Apache-2.0, originally IBM/SAFR)
high-volume reporting and extraction system for IBM z/OS. The **Performance
Engine** is its batch runtime. Business users design *views* (roughly: a
query plus an output layout) in a workbench; the workbench compiles the views
into two machine-readable artefacts, the **VDP** (View Definition Parameters)
and the **logic table**. The Performance Engine reads those artefacts, reads
the business data ("event files") once, applies *all* views to every record in
a single pass, and produces extract files and, after sorting, formatted
reports and output files.

The implementation is HLASM because the design goal is to process very large
event files at close to hardware speed. Rather than interpreting the logic
table for every record, the engine compiles it into z/Architecture machine
code at start-up (see [06-generated-code.md](06-generated-code.md)).

```
                   Workbench (not in this repo)
                            |
              +-------------+--------------+
              |                            |
              v                            v
        VDP (binary records)       Logic table (rows)
              |                            |
              +-------------+--------------+
                            |
      +---------------------v----------------------+
      |   GVBMR95  (alias GVBMR95R / GVBMR95E)     |
      |   +--------------------------------+       |
      |   | GVBMR96: load, validate,       |       |
      |   |          generate machine code |       |
      |   +--------------------------------+       |
      |   threads: read event files, run generated |
      |   code per record, write extract records   |
      +----------------------+---------------------+
                             |
        reference phase:     |    extract phase:
        REH/RED lookup files |    EXTRnnn extract files
                             |
                             v
                 external SORT (DFSORT/SyncSort)
                             |
                             v
      +--------------------------------------------+
      |   GVBMR88 (initialised by GVBMR87)         |
      |   sorted extract -> breaks, totals,        |
      |   formatting (GVBDL96), reports/CSV/XML    |
      +--------------------------------------------+
```

## 1.2 The four processing phases

The same load module `GVBMR95` performs the first two phases; which one runs is
decided by the **alias** used in the JCL `EXEC PGM=` statement (see
[03-gvbmr95-extract-engine.md](03-gvbmr95-extract-engine.md#32-alias-enforcement)).

| Phase | Program (alias) | Input | Output |
|-------|-----------------|-------|--------|
| **Reference** | `GVBMR95R` | Reference-data files described in the VDP; a logic table whose views write reference records | `REH` (reference header) and `RED`/`REFR*` (reference data) files that the extract phase loads into memory for lookups |
| **Extract** | `GVBMR95E` | Event files (BSAM sequential, VSAM KSDS, DB2 SQL, DB2 HPU, Adabas, pipes, tokens), VDP, logic table, REH/RED files | `EXTRnnn` extract files (one per view output DDNAME), control report, log, optional trace |
| **Sort** | external `SORT` | extract files | sorted extract (sort key is a prefix of each extract record) |
| **Format** | `GVBMR88` | sorted extract, VDP, `MR88PARM`, reference tables | reports, fixed-format files, CSV, XML, online blocks |

Source comments in `GVBMR95.asm` describe the extract program as follows
(paraphrased): *"driver" (event) files are read; each record may be processed
against multiple views; lookups are performed against memory-resident tables;
extract records are written to output files.*

## 1.3 Repository layout

```
Performance-Engine/
├── ASM/            24 HLASM programs (*.asm)
├── MAC/            92 HLASM macros / copybooks (*.mac) — DSECTs, equates, code macros
├── TABLE/
│   ├── PGM.csv     program inventory (name, description, type, LOADMOD/OBJONLY, DB2 flag)
│   ├── MAC.csv     macro inventory
│   └── tablesPE.csv
├── LINKPARM/       FreeMarker (.ftl) binder control statements, one per load module
├── JCL/BIND.JCL    DB2 BIND PACKAGE/PLAN job for GVBMRSQ
├── V10727.xml      sample workbench view export (VDP content in XML form)
├── zapp.yaml       IBM Z Open Editor/ZAPP: MAC/ is the HLASM SYSLIB
├── .gitattributes  z/OS encoding rules (EBCDIC working tree, ISO-8859-1 in git)
├── README.md       USS/Git setup notes
└── LICENSE         Apache-2.0
```

There is no build script in the repository; `TABLE/PGM.csv` and
`LINKPARM/*.ftl` are the inputs consumed by the GenevaERS build process
(DBB-style, *(inferred)* from the FreeMarker templates and the CSV column
names `PSRCREPO`, `PSRCFLDR`, `PMODTYPE`).

## 1.4 Program inventory (from `TABLE/PGM.csv`)

| Program | Description (PGM.csv) | Type | Link type | Notes |
|---------|-----------------------|------|-----------|-------|
| `GVBMR95` | View Extract Process | Main Program | LOADMOD | Extract/reference engine; aliases `GVBMR95E`, `GVBMR95R` |
| `GVBMR96` | Initialization for GVBMR95 | Subroutine | OBJONLY | Linked into `GVBMR95`; loads VDP/LT, generates code |
| `GVBMRBS` | BSAM Initialization for GVBMR95 | Subroutine | OBJONLY | Sequential event files, pipes, tokens |
| `GVBMRAD` | Adabas I/O Handler | Subroutine | OBJONLY | Linked into `GVBMR95` |
| `GVBMRSQ` | Db2 I/O Handler | Subroutine | OBJONLY | Linked only when `GERS_DB2_ASM=Y`; needs DB2 precompile (`PDB2PRE=Y`) |
| `GVBMRSU` | Db2 HPU Handler | Subroutine | OBJONLY | Linked only when `GERS_DB2_ASM=Y` |
| `GVBMRVK` | VSAM Keyed I/O Handler | Subroutine | OBJONLY | KSDS event/reference files |
| `GVBSRCHR` | Lookup routines with keylen 1 to 256 | Subroutine | OBJONLY | 256 generated binary-search routines |
| `GVBMRHPU` | Db2 HPU Inzexit | Main Program | LOADMOD | Entry `INZEXIT`, APF `AC(1)`; HPU user exit |
| `GVBMR88` | View Format Process | Main Program | LOADMOD | Format phase |
| `GVBMR87` | Initialization for GVBMR88 | Subroutine | OBJONLY | Linked into `GVBMR88` via `V(GVBMR87)` |
| `GVBDL96` | Enhanced Field Format Process | Subprogram | LOADMOD | Field format/mask conversion; entry `GVBDL96X` (AMODE 64) used by MR95 |
| `GVBDAYS` | Date Difference Calculator | Subprogram | LOADMOD | `CCYYDDD` arithmetic |
| `GVBTP90` | VSAM/QSAM I/O Handler | Subprogram | LOADMOD | Callable from COBOL |
| `GVBUR20` | Sequential I/O Handler | Subprogram | LOADMOD | BSAM disk/tape |
| `GVBUR33` | Symbolic Variable Substitution | Subprogram | LOADMOD | |
| `GVBUR35` | Dynamic Allocation Interface | Subprogram | LOADMOD | SVC 99 |
| `GVBUR39` | GENWRITE Interface | Subprogram | LOADMOD | |
| `GVBURALI` | ALIAS | Subprogram | LOADMOD | Which alias was `GVBMR95` invoked by |
| `GVBURZTM` | CPU time formatter | Subroutine | LOADMOD | zIIP/enclave time report |
| `GVBUT99` | User Abend Utility | Main Program | LOADMOD | |
| `GVBUTHDR` | Header format builder | Subprogram | LOADMOD | |
| `GVBUTMSG` | Message Builder | Subprogram | LOADMOD | |
| `GVBUTMUE` | Mixed-Case Message Table | Subroutine | OBJONLY | Generated from `GVBMSGGE.mac` |

### External modules referenced but not in this repository

`GVBMR95`/`GVBMR96` declare these as weak externals (`WXTRN`) and test the
V-con for zero before using them:

| Symbol | Purpose (from surrounding comments) |
|--------|-------------------------------------|
| `GVBMRZP` | zIIP function module (`ZIIPADDR`; functions `zIIP_init`, `zIIP_oct`, `SRB_sched`, `TCB_switch`, `SRB_switch`, `SRB_end` in `GVBMRZPE.mac`) |
| `GVBMRDV` | "DB2 via VSAM" event reader (access method `DB2VSAM`) |
| `GVBMRDI` | referenced by `GVBMR96` (`MRDIADDR`) |
| `GVBMRCT` | DB2 catalog plan (parameter `DB2_CATALOG_PLAN_NAME`) |
| `GVBXRCK` | "common key read exit" name compared against `LTRENAME` |
| `GENWRITE` | located dynamically by `GVBUR39` |
| `SORT`, `CEEPIPI`, `DSNALI`, `DSNHLI2`, `GENXS88`, `GENXD88` | system sort, LE pre-init (COBOL exits), DB2 call attach, MR88 write exits |

## 1.5 Terminology primer

* **View** — a unit of work designed in the workbench; identified by a numeric
  view ID (e.g. `10727`). One event record can feed many views.
* **VDP** — View Definition Parameters. A file of variable-length records,
  each with a common `vdp_header` (record type 0001, 0050, 0200, 0300, 0400,
  ..., 4200). Layouts are in `MAC/GVBnnnnA.mac`/`GVBnnnnB.mac`.
* **Logic table (LT)** — row-per-instruction description of a view's
  processing. Row types `HD`, `NV`, `RE`, `F0`, `F1`, `F2`, `WR`, `CC`, `EN`,
  `ES`, `ET`, `GEN`. Loaded from DDNAME `EXTRLTBL`.
* **Event file / driver file / source file** — the business data being read
  (the "ES set" in code: "event set").
* **Extract record** — the record `GVBMR95` writes: length prefix, sort key,
  sort title, data (see `EXTREC` in [12-control-blocks.md](12-control-blocks.md)).
* **Reference table / lookup** — a memory-resident table keyed by a lookup
  key, loaded from REH/RED files produced by the reference phase.
* **LR** — Logical Record (a record layout). **LF** — Logical File. **PF** —
  Physical File. Referenced by IDs in the VDP.
* **Thread** — a z/OS sub-task (`ATTACHX`) processing one or more event files.
* **THRDAREA** — the per-thread work area DSECT (`MAC/GVBMR95W.mac`).
* **Literal pool** — a per-view data area (constants, accumulators, counters)
  addressed from generated code via R2.
