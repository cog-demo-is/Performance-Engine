# 15. Glossary

Terms are grouped into z/OS and HLASM vocabulary, GenevaERS runtime
vocabulary, and the names of the internal structures that recur throughout
documents 03-13. Each entry says where the term is defined or used in the
repository so it can be checked against the source. Symbol names are written
exactly as they appear in the assembler listings (HLASM is case-insensitive
for symbols, but the sources mix `LTESCODE` and `ltescode`; documents use the
form that appears at the definition).

---

## 15.1 HLASM and z/OS vocabulary

**HLASM** - IBM High Level Assembler (`ASMA90`). All 24 programs and 92
macros are HLASM. The programs use structured-programming macros
(`IF`/`ELSE`/`ENDIF`, `DO`/`ENDDO`, `SELECT`/`CASENTRY`/`WHEN`) from the
HLASM Toolkit, which is why the README lists the Toolkit as a prerequisite.

**AMODE / RMODE** - addressing mode (24, 31, 64 bits) and residency mode
(where the Binder may load a module: below 16 MB, anywhere below 2 GB, or
`SPLIT`). Declared per CSECT (`GVBMR95 AMODE 31`, `GVBMR96 RMODE ANY`) and
overridden or confirmed at link time by `SETOPT PARM(AMODE=..)`,
`SETOPT PARM(RMODE=..)` in `LINKPARM/*.ftl`. See
[14-build-and-deployment.md](14-build-and-deployment.md).

**SAM24 / SAM31 / SAM64** - *Set Addressing Mode* instructions that switch
the running AMODE. `GVBMR95` and `GVBMR96` issue `SAM64` on entry and `SAM31`
around every call to a 31-bit driver or utility.

**BASSM / BASR / BSM** - branch instructions. `BASSM` (*Branch And Save And
Set Mode*) is used to call 31-bit routines from 64-bit code and to return
through `BSM`; `BASR` is the plain call used between same-mode modules
(`BASR R14,R15`).

**CSECT / DSECT** - control section (code/data that occupies storage
in the object module) and *dummy* section (a
layout template that occupies no storage and is applied to a base register
with `USING`). Almost every `MAC/*.mac` file is a DSECT describing a control
block (see 15.3).

**USING / DROP** - HLASM directives that associate a base register with a
DSECT or location so that symbolic field names resolve to
base-displacement addresses. Misplaced `USING`s are the classic source of
"wrong control block" bugs; the sources use labelled and dependent `USING`s
(`saveit using saver,savesort`) to avoid them.

**ASSERT** - HLASM Toolkit macro that fails the assembly when a compile-time
condition is false. Used to pin layouts that other code depends on
(`ASSERT stdparml,eq,execdlen` in `GVBMR96`,
`ASSERT (ltwr_next_exit-logictbl),EQ,(ltre_next_exit-logictbl)` in
`GVBMR95L.mac`).

**Reentrant (`REUS=RENT`)** - a module that never modifies its own storage
and can therefore be shared by several tasks simultaneously. Required for
utilities called from the parallel extract threads. `REUS=NONE` modules
(`GVBMR88`, `GVBUTMSG`, ...) are not shared.

**Literal / literal pool (HLASM sense)** - `=V(GVBUTMSG)`, `=CL8'GVBMR95R'`
constants collected by the assembler at `LTORG`. Not to be confused with the
runtime *literal pool* built by `GVBMR96` (15.3).

**V-constant / WXTRN / EXTRN** - `DC V(name)` creates an external reference
that the Binder must resolve. `EXTRN`/strong `V` references fail the link if
unresolved; `WXTRN` (*weak external*) references resolve to zero when the
member is absent, which is how `GVBMR95` tests for optional drivers
(`WXTRN GVBMRZP`, `WXTRN GVBMRSQ`, `WXTRN GVBMRSU`, `WXTRN GVBMRAD`,
`WXTRN GVBMRDV`).

**Binder (`IEWL`, `IEWBLINK`)** - the z/OS program that turns object decks
into load modules/program objects, driven by control statements
(`INCLUDE`, `ENTRY`, `ALIAS`, `NAME`, `SETOPT`). The `LINKPARM/*.ftl`
templates generate these statements.

**Alias** - an additional member name for a load module. `GVBMR95` is bound
with `ALIAS GVBMR95E,GVBMR95R`; the program detects which name it was
invoked under (`GVBURALI`, `namepgm`) and selects the extract or reference
phase, refusing the base name (`EXEC_PGM_ERR`).

**Load module / program object / object deck (`OBJONLY`)** - the executable
member in a `LOADLIB`, versus the assembler output that is only ever bound
into another module. `TABLE/PGM.csv` uses `LOADMOD` and `OBJONLY`.

**APF authorization / `TESTAUTH` / `MODESET`** - Authorized Program Facility.
Modules from an APF-authorized library with `AC(1)` may issue restricted
services. `TESTAUTH FCTN=1` tests whether the step is authorized;
`MODESET KEY=ZERO,MODE=SUP` switches to key 0/supervisor state (zIIP path).
Required for `PAGE_FIX_IO_BUFFERS=Y`, `USE_ZIIP=Y` and DB2 HPU.

**TCB / SRB** - Task Control Block (a dispatchable task created by
`ATTACH`/`ATTACHX`) and Service Request Block (a lighter dispatchable unit
that may run on a zIIP). The extract threads are TCBs; with `USE_ZIIP=Y`
the generated-code phase of each thread runs under an SRB and switches back
to the TCB for I/O (`TCB_switch`/`SRB_switch` in `MAC/GVBMRZPE.mac`).

**zIIP** - IBM z Integrated Information Processor, a specialty engine to
which eligible SRB work (in an enclave) can be dispatched. Support is in the
external module `GVBMRZP`.

**WLM / enclave** - Workload Manager and the enclave services
(`IWM4ECRE`, `IWMEJOIN`, `IWMELEAV`, `IWM4EDEL`, `IWMEQTME`) that make SRB
work zIIP-eligible and account its CPU time.

**Pause elements (`IEA4APE`, `IEA4PSE`, `IEA4XFR`, `IEA4RLS`, `IEA4DPE`)** -
z/OS services to allocate, pause on, transfer to, release and deallocate a
pause element; used to hand control between the TCB and SRB halves of a
zIIP-enabled thread.

**ECB / `WAIT` / `EVENTS` / `POST`** - Event Control Block and the services
to wait on and post it. The mother task waits on the daughter ECBs with
`EVENTS`; each `ATTACHX ... ECB=(7)` names the ECB posted at thread end.

**`STORAGE OBTAIN` / `IARV64`** - 31-bit and 64-bit storage acquisition.
Thread areas, logic tables and I/O buffers are `STORAGE OBTAIN`ed
(`CHECKZERO=YES` avoids clearing already-zero pages); reference-data pools
and the hash index use `IARV64 REQUEST=GETSTOR` above the 2 GB bar.

**ESTAE / ESTAEX / SDWA / retry** - recovery environment. Each thread issues
`ESTAEX (r3),CT,PARAM=(R2),PURGE=HALT`; on an abend z/OS drives the exit
(`TASKABND`) with a System Diagnostic Work Area describing the failure.
Controlled by `RECOVER_FROM_ABEND`.

**Program mask / `SPM`** - PSW bits that decide whether fixed-point and
decimal overflow cause a program interruption. `ABEND_ON_CALCULATION_OVERFLOW`
builds `ovflmask` (`X'0C000000'`) which each thread loads with `SPM`.

**BSAM / QSAM / VSAM (KSDS, ESDS) / EXCP** - z/OS access methods. `GVBMRBS`
and `GVBUR20` use BSAM (`READ`/`WRITE`/`CHECK` with DECBs); `GVBTP90`
handles VSAM and QSAM; `GVBMRVK` reads keyed VSAM (KSDS). Access-method
codes in the VDP (`SEQFILE=1`, `VSAMFILE=2`, `KSDSFILE=3`, `EXCPFILE=8`,
...) select the driver.

**DCB / DCBE / ACB / RPL / DECB** - Data Control Block (sequential file),
its 31-bit extension (`DCBE`, which carries `EODAD`, `SYNAD`, `MULTSDN`,
`RMODE31=BUFF`), the VSAM Access method Control Block and Request Parameter
List, and the Data Event Control Block that tracks one BSAM `READ`/`WRITE`.
`GVBMR95` stores `EVNTEOF`/`SYNADEX0` into the DCBE before dispatching a
driver.

**RDW / RECFM F, V, VB / LRECL / BLKSIZE** - record descriptor word (4-byte
length prefix on variable records) and the record-format vocabulary that
`GENFILE` mirrors (`GPRECFMT`, `GPRECLEN`, `GPRECMAX`, `GPBLKMAX`).

**DDNAME / TIOT / SVC 99 (`DYNALLOC`)** - the JCL name of a data set, the
Task I/O Table that lists the DDs present in the step, and dynamic
allocation. `GVBUR35` issues SVC 99; `TREAT_MISSING_VIEW_OUTPUTS_AS_DUMMY`
checks the TIOT for extract DDs.

**SNAP / WTO / `LOGIT`** - `SNAP` dumps storage to a data set
(`EXTRDUMP`/`SNAPDATA`), `WTO` writes to the operator console/job log, and
`LOGIT` is the local macro that writes a line to the control report or log.

**SORT, E15 / E35** - DFSORT/SyncSort and its input (E15) and output (E35)
user exits. `GVBMR88` runs as the E35 exit of the sort that orders the
extract file (`SORTRSA`, `E35RETRN`, `SORTE35`).

**CAF (`DSNALI`) / DBRM / plan / package / bind** - DB2 Call Attach
Facility used by `GVBMRSQ` to `CONNECT`/`OPEN`; the Database Request Module
produced by the DB2 precompiler; and the bind artefacts created by
`JCL/BIND.JCL` (`BIND PACKAGE`, `BIND PLAN(GVBMRSQ&SPLANSFX)`).

**DB2 HPU / `INZEXIT`** - IBM Db2 High Performance Unload and the
initialization exit (`GVBMRHPU`, entry `INZEXIT`) through which unloaded rows
are handed to the engine (`GVBMRSU`).

**Adabas** - Software AG database accessed by `GVBMRAD` (access method
`CALLADA=17`) using direct commands (`L3` block reads) with `DBID`, `FNR`,
`FB`, `SB` parameters.

**LE (Language Environment)** - IBM's common runtime for COBOL/C/PL/I.
Exits written in LE languages are described by the `LEINTER` block
(`MAC/GVBMR95C.mac`, `MAC/GVBMR88C.mac`) and typed through
`LTWRPGM_TYPE_LECOBOL`, `_COBOL2`, `_C`, `_CPP`, `_JAVA`, `_ASM` so the HLASM
threads can build the right linkage before calling them.

**EBCDIC / IBM-1047 / ISO8859-1** - the host code page for the sources on
z/OS and the code page in which Git stores them; mapped by `.gitattributes`.

---

## 15.2 GenevaERS runtime vocabulary

**GenevaERS / Performance Engine** - the open-source (Apache 2.0) reporting
engine whose z/OS runtime this repository contains. The *Workbench* (not in
this repository) designs views; the *compiler* produces the VDP and logic
table; the Performance Engine executes them.

**Phase** - one job step of a run. The four phases documented here:

```text
 reference (GVBMR95R) -> extract (GVBMR95E) -> SORT -> format (GVBMR88, init GVBMR87)
```

**View** - a single report/extract definition. Identified by a view number
(`LTVIEW#`, `vdp_VIEW_no`). A run processes many views in one pass over the
event data; `RUNVIEWS` restricts the set.

**VDP (View Definition Parameters)** - the binary file (`MR95VDP`,
`MR88VDP`) produced by the compiler and read by `GVBMR96`/`GVBMR87`. Records
are typed (`vdp_RECORD_TYPE`: 0001 generation, 0200 physical file, 0650
lookup path, 0801 ..., 1000 view, ...) and share the `vdp_header`
prefix. See [12-control-blocks.md](12-control-blocks.md).

**Logic table (LT), JLT, XLT** - the compiled program for a run: a sequence
of rows each carrying a four-character function code. `JLT` (join logic
table) drives the reference phase (`GVBMR95R`) and `XLT` (extract logic
table) drives the extract phase (`GVBMR95E`); both arrive through the
`EXTRLTBL`/`REFRLTBL` DD and are validated against the VDP timestamp
(`VERIFY_CREATION_TIMESTAMP`). See [05-logic-table.md](05-logic-table.md).

**Function code (`LTFUNC`)** - the 4-character operation of a logic-table
row, e.g. `NV`, `RENX`, `LKE`, `LUSM`, `CFEC`, `DTE`, `SKC`, `WRXT`, `ES`.
The first letters are the *major function*, the rest name operand sources
(`C` constant, `E` event, `L` lookup, `P` previous, `X` prior column, `A`
accumulator). Matched against `GVBmaj_t` and `FUNCTBL`.

**Event file / event record** - the input data being extracted (sequential,
VSAM, DB2, Adabas, pipe or token). One *ES set* reads one event file.

**ES set (event set)** - the rows from an `RE` row to its matching `ES` row:
all views that read the same event file, cloned per thread. The unit of
parallelism; a thread picks ES sets from the `LTNXDISK`/`LTNXTAPE`/
`LTNXOTHR` queues.

**NV (new view) / `NVPROLOG`** - the row that starts a view inside an ES
set; the generated prologue that initializes the view's extract record and
accumulators.

**Reference data / lookup / reference table** - data joined to the event
record by key. Produced by the reference phase as `REH` (reference header,
`REFRREH`/`EXTRREH`) and `RED`/`GREF` (reference data, `GREFxxx` DDs) files,
loaded into memory as `LKUPBUFR`/`LKUPTBL` and searched by generated binary
search (`GVBSRCHR`) or hash (`HASH_PACK`, `HASH_MULT`).

**LR (logical record) / LR id** - the compiler's description of a record
layout; `LTLULRID`/`vdp_..._LRID` identify which layout a lookup returns.

**Lookup path (VDP 0650) / join step / JOIN** - a possibly multi-step
navigation from event record to reference record(s). Each step builds a key
(`LK*` rows), performs the search (`LUSM`/`LUEX`) and may feed the next step
(`JOIN`).

**Effective date / as-of date** - reference tables may be effective-dated;
`LKDC`/`LKDE` rows supply the date and the search returns the row in effect
(`RUN_DATE`, `FISCAL_DATE_*`, `exec_rdate`, `exec_fdate`).

**Extract record / extract file (`EXTRnnn`) / DT and CT areas** - the
output of the extract phase: a sort key (`SK*`), data columns (`DT*`) and
calculated columns (`CT*`) written by `WR*` rows to the `EXTFILE`
controlled by DD `EXTRnnn`.

**Token / pipe** - in-memory record hand-off between views (`WRTK`/`RETK`
token rows) or between ES sets through a pipe (`LTPIPELS`, `PIPE_WARN`),
avoiding intermediate files.

**Exit (read / lookup / write / format)** - user programs invoked at defined
points: read exits (`REEX`, `LTREEXIT`), lookup exits (`LUEX`, `LTLUEXIT`),
write exits (`WREX`, `LTWRNAME`/`LTWRADDR`), format exits in `GVBMR88`. Lookup exits
return 0 found / 4 not found / 8 skip record / 12 disable view / 16+ abort.

**Format phase / sort key break / subtotal / grand total** - `GVBMR88`
consumes the sorted extract, detects breaks in the sort key, accumulates
subtotals per break level and grand totals, and writes report lines or
files. See [10-format-phase-gvbmr87-gvbmr88.md](10-format-phase-gvbmr87-gvbmr88.md).

**Control report** - the human-readable summary each phase writes
(`EXTRRPT`, `REFRRPT`, `MR88RPT`): parameters, files, record counts,
timing (`GVBURZTM`), and messages.

**Trace** - `TRACE=Y` plus `EXTRTPRM` rows make generated code call
`MR95TRAC` (`ENTRY MR95TRAC`; `MR95TRAC BAS R10,TRACE`) for selected
views/rows/records, writing to `EXTRTRAC`; without trace the call site is a
`NOP`.

**Execution parameters (`EXTRPARM`, `REFRPARM`, `MR88PARM`)** - keyword
files parsed at start (`PARMKWRD_table` in `GVBMR96`) into `EXECDATA` and the
format `WORKAREA`. See [13-parameters-ddnames-messages.md](13-parameters-ddnames-messages.md).

**Message number / `GVBMSG` / `GVBUTMSG` / `GVBUTMUE`** - every diagnostic
has a symbolic number in `MAC/GVBUTEQU.mac` (`THREAD_ABEND EQU 9`, ...);
the `GVBMSG` macro builds a `GENMSG` parameter list, `GVBUTMSG` formats the
text from the table assembled in `GVBUTMUE` (`GVBMSGGE`), and writes it
(`LOG`, `WTO`, `FORMAT`).

**Return code / user abend (`Unnnn`) / system abend (`Sxxx`)** - the step
completion code (0, 1 for empty logic table, 4, 8, 16) versus abends forced
by the engine (`ABEND 999`, `GVBUT99`, `DC XL4'FFFFFFFF'` = S0C1) or by
hardware (S0C4, S0C7, S0C8, S0CA).

---

## 15.3 Internal structures (see [12-control-blocks.md](12-control-blocks.md))

| Symbol | Defined in | What it is |
|---|---|---|
| `THRDAREA` | `MAC/GVBMR95W.mac` | per-thread work area: registers, ECB, current ES row, counters, `THRDEXEC` chain, `THRDTYP`, `abend_msg`, `abend_lt`, `ovflmask` |
| `EXECDATA` | `MAC/EXECDATA.mac` | parsed execution parameters, initialized from `STDPARMS` |
| `PARMTBL` | `MAC/GVBMR95C.mac` | one chained entry per `TRACE` request (`PARMVIEW`, `PARMFROM`, ...) |
| `LOGICTBL` / `LTREDEFN` | `MAC/GVBMR95L.mac` | in-memory logic-table row and its function-specific overlays (`LTRE*`, `LTES*`, `LTLU*`, `LTWR*`, `LTNV*`) |
| `LTF0_REC`, `LTF1_REC`, `LTF2_REC` | `MAC/GVBLTF0A.mac`, `GVBLTF1A.mac`, `GVBLTF2A.mac` (plus `GVBLTHDA`, `GVBLTNVA`, `GVBLTREA`, `GVBLTWRA`, `GVBLTCCA`, `GVBLTV*A`) | external logic-table record layouts |
| `FUNCTBL` / `GVBFUNTB` / `GVBmaj_t` | `ASM/GVBMR95.asm`, `MAC/MAJRFTAB.mac` | function-code table: record type, code/pool lengths, model and relocation addresses |
| `MDLxxxx`, `MDLxxxxL/P/R` | `MACHCODE` CSECT in `ASM/GVBMR95.asm` | model code skeletons, their lengths, pool usage and relocation tables |
| `CS*` codes (`CSSRCLN`, `CSTRUEO`, ...) | `ASM/GVBMR95.asm` | relocation types applied by PASS1/PASS2 |
| `LITP_HDR` / literal pool | `MAC/GVBMR95C.mac` | per-view runtime constant area addressed by R2 from generated code |
| `NVPROLOG` | `MAC/GVBMR95C.mac` | header of the generated per-view prologue |
| `callview_dsect` | `MAC/GVBMR95C.mac` | parameter block for view-to-view (`CSCALLVW`) calls |
| `EXTFILE` / `EXTREC` | `MAC/GVBMR95C.mac` (`EXTREC` also in `GVBMR88C.mac`) | extract output file control block and record header |
| `LTWRAREA` / `SUMAREA` / `STACKENT` | `MAC/GVBMR95C.mac` | write-row work area, summarisation buffer, join/exception stack entry |
| `GENPARM` / `GENENV` / `GENFILE` / `GENEXTR` | `MAC/GVBX95PA.mac` (`GENPARM` also in `GVBAX88P.mac`) | the common parameter list passed to I/O drivers and exits (environment, file, extract record) |
| `exit_data`, `EXHEXH`, `EXUEXU`, `LEINTER` | `MAC/GVBMR95C.mac` (`LEINTER` also in `GVBMR88C.mac`) | exit interface blocks (header, user area, LE bridge) |
| `INITVAR` | `MAC/GVBMR95C.mac` | initial-value block for `DIM` variables |
| `TBLHEADR` | `MAC/GVBMR95C.mac`, `MAC/GVBMR88C.mac` | reference-table header record (REH/RTH) |
| `LKUPBUFR` / `LKUPTBL` | `MAC/GVBMR95C.mac`, `MAC/GVBMR88C.mac` | lookup-buffer prefix and table-entry prefix in the reference pool |
| `vdp_header`, `VDP0001_GENERATION_RECORD`, `VDP0650_JOIN_RECORD`, ... | `MAC/VDPHEADR.mac`, `MAC/GVBnnnnA.mac` / `GVBnnnnB.mac` | VDP record layouts (one macro pair per record type: 0001, 0002, 0050, 0100, 0200, 0210, 0300, 0400, 0500, 0600, 0601, 0650, 0700, 0800, 0801, 1000, 1200, 1210, 1300, 1400, 1600, 2000, 2200, 2210, 2300, 3000, 3200, 4000, 4200) |
| `WORKAREA` (format) / `GENRPT` / `GENRUN` | `MAC/GVBMR88W.mac`, `MAC/GVBAX88P.mac` | `GVBMR88` work area, report descriptor, run descriptor |
| `GENMSG` | `MAC/GVBMSG.mac` (`GVBMSG DSECT`) | message-builder parameter list |
| `M35SVC99` | `MAC/GVBAUR35.mac` | `GVBUR35` dynamic-allocation parameter block |
| `UR20PARM` | `MAC/GVBUR20P.mac` | `GVBUR20` parameter block |
| `DL96AREA`, `DL96EQU` | `MAC/DL96AREA.mac`, `MAC/DL96EQU.mac` | `GVBDL96` work area and equates |
| `EXTXPLST` | `ASM/GVBMRHPU.asm` | DB2 HPU exit parameter list |

---

## 15.4 Register conventions in one place

```text
 generated code / GVBMR95                       GVBMR88
 ────────────────────────                       ───────
 R13  THRDAREA                                  R13  WORKAREA
 R12  GVBMR95 base (V-con access)               R12  program base
 R11  NVCONST of the current view               R11  -
 R2   literal pool address minus 512K           R2   work
 R6   current event record                      R6   current SORTKEY / COLDEFN / LKUPBUFR
 R7   extract record (EXTREC)                   R7   current extract record
 R8   current extract column                    R8   current VIEWREC
 R5   current reference record / lookup buffer  R5   loop counter / reference record
 R3   previous record for ..P operands          R3, R4 work
 R9/R10 subroutine return (2nd / 1st level)     R9/R10 subroutine return
 R14, R15, R0, R1  linkage / work               R14, R15, R0, R1  linkage / work
```

In the `GVBMR95` mother-task and thread-control code (outside generated
code) R8 addresses the current `LOGICTBL` row (`USING LOGICTBL,R8`).

(Exact usage per routine is in the prologue comments of each program;
[12-control-blocks.md](12-control-blocks.md) has the full recap.)

---

## 15.5 Abbreviations used in file and DD names

| Prefix / DD | Meaning |
|---|---|
| `GVB` | GenevaERS module prefix (all programs and macros) |
| `MR` | "main routine" programs (`GVBMR95`, `GVBMR96`, `GVBMR87`, `GVBMR88`, `GVBMRxx` drivers) |
| `UR` | utility routines (`GVBUR20`, `GVBUR33`, `GVBUR35`, `GVBUR39`, `GVBURALI`, `GVBURZTM`) |
| `UT` | utility programs (`GVBUT99`, `GVBUTHDR`, `GVBUTMSG`, `GVBUTMUE`) |
| `DL` | data/format library (`GVBDL96`) |
| `TP` | file interface (`GVBTP90`) |
| `EXTR*` | extract-phase DDs: `EXTRPARM`, `EXTRENVV`, `EXTRTPRM`, `EXTRLTBL`, `EXTRREH`, `EXTRRPT`, `EXTRLOG`, `EXTRTRAC`, `EXTRDUMP`, `EXTRnnn` |
| `REFR*` | reference-phase DDs: `REFRPARM`, `REFRENVV`, `REFRTPRM`, `REFRLTBL`, `REFRREH`, `REFRRPT`, `REFRLOG`, `REFRTRAC`, `REFRDUMP`, `REFRnnn`; `REFRRTH` is the header file read by `GVBMR87` |
| `MR95VDP` / `MR88VDP` | VDP input for extract/reference and format phases |
| `MR88PARM`, `MR88HXE`, `MR88RTD`, `MR88DATA`, `MR88PRNT`, `MR88RPT`, `MR88LOG` | format-phase parameters, sorted extract, reference data, data output, print output, report, log |
| `GREFnnn` | reference-data files (VDP 0650 `grefcnt`) |
| `RUNVIEWS` | optional list of view numbers to run |
| `Hnnnnnnn` | hash statistics report opened when `DISPLAY_HASH=Y` |
| `SORT` / `SYSIN` | sort control statements written by `GVBMR95` / read by `GVBMR87` when `GVBMR88` invokes SORT |
| `SNAPDATA` | `SNAP` output DD used by `GVBMR88` |

The full DD/DCB tables are in
[13-parameters-ddnames-messages.md](13-parameters-ddnames-messages.md).
