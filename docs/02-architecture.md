# 2. Architecture

## 2.1 Component map

```
 ┌─────────────────────────────── load module GVBMR95 (RMODE=SPLIT) ───────────────────────────────┐
 │  ENTRY GVBMR95   ALIAS GVBMR95E (extract)  ALIAS GVBMR95R (reference)                            │
 │                                                                                                  │
 │  GVBMR95.asm  main task + sub-task entry MR95THRD + model code (MACHCODE csect) + FUNCTBL        │
 │      │                                                                                           │
 │      ├─► GVBMR96.asm   init: parms, VDP, logic table, lookups, threads, code generation (PASS1)  │
 │      │        └─► GVBUR33 (symbolics)  GVBUR35 (SVC99)  GVBUTMSG  GVBSRCHR  GVBMRVK (ref load)   │
 │      │                                                                                           │
 │      ├─► event readers (selected per event file):                                                │
 │      │      GVBMRBS (BSAM seq/pipe/token)  GVBMRVK (KSDS)  GVBMRSQ (DB2 SQL)                     │
 │      │      GVBMRSU (DB2 HPU)  GVBMRAD (Adabas)  GVBMRDV (external, DB2/VSAM)                    │
 │      │                                                                                           │
 │      ├─► GVBMRZP (external, weak) zIIP/SRB services                                              │
 │      ├─► GVBDL96X (AMODE 64 entry of GVBDL96) field formatting from generated code               │
 │      ├─► GVBUTMSG / GVBUTMUE  messages;  GVBUTHDR report headers;  GVBURZTM CPU/zIIP timing      │
 │      └─► GVBURALI  which alias am I?                                                             │
 └──────────────────────────────────────────────────────────────────────────────────────────────────┘

 ┌────────────── load module GVBMR88 (AMODE 31, RMODE 24) ──────────────┐
 │  GVBMR88.asm  format phase mainline                                   │
 │      ├─► GVBMR87.asm  init: MR88PARM, VDP, ref tables, DCBs, sort     │
 │      ├─► GVBDL96 field formatting     GVBDAYS  date arithmetic        │
 │      ├─► GVBTP90 VSAM/QSAM I/O        GVBUR20  BSAM I/O               │
 │      ├─► GVBUTMSG messages            GVBUTHDR headers                │
 │      └─► SORT (LINK) with E15/E35 exits when SORT_EXTRACT_FILE is set │
 └───────────────────────────────────────────────────────────────────────┘
```

`LINKPARM/GVBMR95.ftl` is the authoritative list of what is bound into the
extract load module:

```
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

`GVBMRBS`, `GVBMRVK`, `GVBSRCHR`, `GVBUTMUE` etc. are pulled in by binder
autocall through `V(...)` address constants in `GVBMR95`/`GVBMR96`.
`GVBMRSQ`/`GVBMRSU`/`GVBMRAD`/`GVBMRDV`/`GVBMRZP` are `WXTRN` (weak) so the
module links cleanly when they are absent; `GVBMR95` tests the V-con for zero
at run time and reports `IO_DRIVER_UNAVAILABLE` if the access method is
requested.

## 2.2 Addressing modes

| Module | Source directives | Notes |
|--------|-------------------|-------|
| `GVBMR95` | `RMODE 31`, `AMODE 31`; executes `sam64` shortly after entry | Most of the mainline, thread code and generated code run in **AMODE 64** (`llgt`, `stg`, `lg`, 64-bit R13 into `THRDAREA`). It drops to `sam31` around 31-bit services (`EVENTS`, `IWM*`, `ATTACHX`, DCB I/O) and back with `sam64`. |
| `MACHCODE` (model code csect inside `GVBMR95.asm`) | `RMODE 31`, `AMODE 31` | Skeletons are copied into generated-code buffers; they are executed in the caller's AMODE 64. |
| `GVBMR96` | `RMODE ANY`, `AMODE 31` | Header: "GVBMR96 runs in 31-bit addressing mode". Called via `bassm` from MR95 (which is in AMODE 64 at that point). |
| `GVBMR87`, `GVBMR88` | `RMODE 31`, `AMODE 31` | Link deck forces `RMODE=24` for the load module (DCBs below the line). |
| `GVBDL96` | AMODE 31 with `ENTRY GVBDL96X` `AMODE 64` | MR95 generated code calls the 64-bit entry; MR88 calls the 31-bit one. |
| `GVBSRCHR` | `RMODE ANY`, `AMODE 31` | Contains 64-bit instructions (`ltg`, `lg`, `stg`); addressed through `THRDAREA` with R13. |

## 2.3 Data flow and DDNAME contracts

```
  EXTRPARM ─┐                                           ┌─► EXTRRPT   control report (VB 164)
  EXTRENVV ─┤                                           ├─► EXTRLOG   log (VB 164)
  EXTRTPRM ─┤   ┌──────────────┐    ┌───────────────┐   ├─► EXTRTRAC  trace (FBA 161), optional
  MR95VDP  ─┼──►│   GVBMR96    │───►│    GVBMR95    │───┼─► EXTRDUMP  SNAP output
  EXTRLTBL ─┤   │  (init/gen)  │    │  (threads)    │   ├─► Hnnnnnnn  hash statistics (DISPLAY_HASH)
  EXTRREH  ─┤   └──────────────┘    └───────┬───────┘   └─► <ltwrddna> extract files, e.g. EXTRnnn
  RUNVIEWS ─┘  (optional view subset)       │                (VB, LRECL 8192 template "EXTR")
                                            │
       event files (DDNAME from VDP 0200) ──┘   REF tables: REH header + one data file per table

  Reference phase uses the same DDNAMEs with prefix REFR instead of EXTR:
  REFRPARM REFRENVV REFRTPRM REFRLTBL REFRREH REFRRPT REFRLOG REFRTRAC REFRDUMP
```

The `EXTR`/`REFR` prefix is substituted at run time based on the alias:
`GVBMR96` copies `=cl8'REFRREH'` etc. over the DCB DDNAME when running as
`GVBMR95R` (comment in source: `default is EXTRREH`).

Format phase:

```
  MR88PARM  ─┐                                 ┌─► report DDNAMEs from VDP (print, file, CSV, XML)
  VDP       ─┼──► GVBMR87 ──► GVBMR88 ─────────┼─► control report
  EXTRnnn   ─┤   (sorted, or sorted in-flight  └─► log
  REH/RED   ─┘    via SORT with E15/E35)
```

## 2.4 Inter-module calling convention

All modules use standard OS linkage (R1 → parameter list, R13 → save area,
R14 return, R15 entry/return code). The key shared parameter block is
**`GENPARM`** (`MAC/GVBX95PA.mac`), which is what `GVBMR95` passes to the
event-file readers and to user exits:

| Field | Meaning (from macro comments) |
|-------|-------------------------------|
| `GPENVA` | → `GENENV` environment information (phase, thread number, view ID, ...) |
| `GPFILEA` | → `GENFILE` file information (DDNAME, RECFM, LRECL, buffer addresses, `GP_redrive` flag) |
| `GPSTARTA` | → start-up data (exit parameter string) |
| `GPEVENTA` | → *pointer to* the current event record (`RECADDR`) |
| `GPEXTRA` | → extract record work area (`EXTREC`) |
| `GPKEYA` | → lookup key |
| `GPWORKA` | → exit work-area pointer (exit-owned storage anchor) |
| `GPRTNCA` | → return code |
| `GPBLOCKA` | → output block pointer |
| `GPBLKSIZ` | → output block size |
| `GENPARM1..5` | additional parameters for utilities/exits |

Within the extract engine, a second convention exists for **generated code**,
documented in the `GVBMR95.asm` header:

```
R13 → THRDAREA (thread work area)          R2  → view literal pool (+512K bias, see 3.7)
R6  → current event record                 R7  → extract record (EXTREC)
R8  → current logic-table row / extract column   R5  → current reference record (lookup)
R3  → previous record pointer (PREVRECA)   R4  → work
R9  → 2nd-level subroutine return          R10/R11 → lookup return / literal addressing
R14/R15 → linkage / work                   R0/R1 → work / parameter list
```

Generated code is entered via `BR R15` where `R15 = LTESCODE` (address of the
generated code for the event-set `ES` row), and returns to the reader loop by
branching to an address planted in the code by `GVBMR96`.

## 2.5 Memory model

* **Main work area** — `STORAGE OBTAIN LENGTH=THRDLEN+l'maineyeb,COND=NO,CHECKZERO=YES`
  in `GVBMR95`; this is the *main* `THRDAREA`, pointed to by R13 and by
  `THRDMAIN` in every thread area.
* **Thread work areas** — one `THRDAREA` per thread, chained by `THRDNEXT`
  from `THRDFRST`; built by `GVBMR96 THRDBLD`. Each thread has its own save
  areas, ECB (`TASKECB`), TCB address (`TCBADDR`), ESTAE state, zIIP state,
  I/O DECB lists, counters.
* **Logic table** — array of `LOGICTBL` rows (variable length, `LTROWLEN`),
  allocated by `GVBMR96 LTBLLOAD`, then **cloned** per event file partition by
  `CLONLTBL` so that each thread has its own copy of the rows it executes.
* **Generated code buffers** — allocated by `GVBMR96 MEMALLOC`, filled by
  `PASS1`, addresses stored in `LTCODSEG`/`LTESCODE`.
* **Literal pools** — one per view (`LITP_HDR` header + literals + accumulators),
  allocated by `ALLOCLIT`; the address kept in `THRDLITP`.
* **Lookup buffers** — `LKUPBUFR` prefix + in-memory table (`LKUPTBL` entries),
  allocated by `LOADLKUP` after reading the REH file for sizes.
* **Event I/O buffers** — DECB + buffer pairs (`GVBDECB.mac`), count from
  `IO_BUFFER_LEVEL`; optionally page-fixed (`PAGE_FIX_IO_BUFFERS`).

## 2.6 Messages and diagnostics

Every message is issued through the `GVBMSG` macro (`MAC/GVBMSG.mac`), which
builds a parameter list and calls `GVBUTMSG` (`V(GVBUTMSG)`) with a message
number (`MAC/GVBUTEQU.mac` equates such as `IDENTIFY_FAIL EQU 1`) and up to
`n` substitution strings. `GVBMSG WTO` also writes to the operator; `GVBMSG
LOG` writes to the log DCB. Message text lives in `MAC/GVBMSGGE.mac` (352
`GVBMSGDF` definitions) which is expanded into the object `GVBUTMUE`.

Diagnostics available at run time: `TRACE` (EXTRTPRM controls which view/row/
record range is traced to `EXTRTRAC`), `DUMP_LT_AND_GENERATED_CODE`,
`INCLUDE_REF_TABLES_IN_SYSTEM_DUMP`, `DISPLAY_HASH`, `SNAP` to `EXTRDUMP`,
and `LOG_MESSAGE_LEVEL`.
