# 13. Parameters, DDNAMEs and messages

This document is the operational reference for running the Performance
Engine: every runtime keyword the programs actually parse, the exact
`EXECDATA` field each keyword lands in, the DDNAMEs each program opens, the
return codes, and the structure of the message catalogue. Everything below is
taken from `ASM/GVBMR96.asm` (`PARMLOAD`), `ASM/GVBMR87.asm` (`READPARM`),
`MAC/EXECDATA.mac`, `MAC/GVBMR95C.mac`, `MAC/GVBUTEQU.mac`, `MAC/GVBMSG.mac`,
`MAC/GVBMSGDF.mac` and `MAC/GVBMSGGE.mac`.

```
  EXTRPARM ──► PARMLOAD ──► PARMKWRD_table (CL35 keyword, H index) ──► casentry ──► EXECDATA field
  EXTRTPRM ──► trace reader ──► TraceParmtable ──► PARMTBL entries (chained, one per TRACE request)
  RUNVIEWS ──► open_runviews ──► runview_list (ascending view numbers) ──► PASS2 disables other views
  MR88PARM ──► READPARM (GVBMR87) ──► parmk0nv table ──► WORKAREA fields (EXTROPT, svrundt, ...)
```

## 13.1 Extract/reference phase parameters (`EXTRPARM` / `REFRPARM`)

### 13.1.1 How the file is parsed

`PARMLOAD` in `GVBMR96` reads the parameter file with `GET` (DCB `PARMDCB`,
`DDNAME=EXTRPARM`; the DDNAME is overwritten with `REFRPARM` when the load
module was entered through the `GVBMR95R` alias). Each record is tokenised
as `KEYWORD=VALUE`; the keyword is compared (`EX R15,PARMCLC` →
`CLC 0(0,R4),parmkwrd`) against the 35-byte entries of `PARMKWRD_table`, and
the 2-byte index that follows the matching keyword selects a `casentry`
branch. Unknown keywords raise `PARM_ERR` (33). The values are validated per
keyword; most switches accept only `Y`/`N` and otherwise raise `PARM_ERR`.

The default values live in `STDPARMS`, a constant block that is copied over
the `EXECDATA` area before the parameter file is read. The two must be the
same size, and the source enforces that at assembly time:

```asm
STDPARML EQU   *-STDPARMS
         ASSERT stdparml,eq,execdlen
```

### 13.1.2 Keyword table

The order below is the order of `PARMKWRD_table` (the source comment says
*Keep in alpha order for the log output*; the five `HASH_*`/`UTILITY`
entries were appended later and break that order). "Case" is the `casentry`
index in `PARMLOAD`.

| Keyword | Case | `EXECDATA` field | Default (`STDPARMS`) | Accepted values / validation |
|---|---:|---|---|---|
| `ABEND_ON_CALCULATION_OVERFLOW` | 14 | `execovfl_on` (CL1) | `Y` | `Y`/`N` (either case), checked in `EDITPARM` (`CALC_OVERFLOW_PARM_ERR`); `Y` sets `ovflmask` to `X'0C000000'` (fixed-point and decimal overflow program-mask bits), which each thread loads with `SPM` when `RECOVER_FROM_ABEND=Y`, so an overflow becomes S0C8/S0CA handled by `TASKABND` |
| `ABEND_ON_ERROR_CONDITION` | 25 | `exec_uabend` (CL1) | `N` | `Y`/`N`; `Y` makes `GVBMR95` issue `ABEND 999` on the `Return_err` path instead of RC 8 |
| `ABEND_ON_LOGIC_TABLE_ROW_NBR` | 20 | `EXECltab` (CL8, echo) and binary `abend_lt` (`GVBMR95W`) | `00000000` | numeric, right-justified and zero-filled (`TRT` with `TRTTBLU`), `PACK`/`CVB` into `abend_lt`; tested in the trace routine (`TRACABND`), so it only fires when `TRACE=Y` and trace parameters exist (`parmtbla` non-zero) |
| `ABEND_ON_MESSAGE_NBR` | 19 | `EXECmsgab` (CL5, echo) and binary `abend_msg` (`GVBMR95W`) | `00000` | numeric, right-justified and zero-filled, `CVB` into `abend_msg`; `ERRMSG#` (`GVBMR95`) and `RTNERROR` (`GVBMR96`) compare R14 with it and execute `DC XL4'FFFFFFFF'` (S0C1) on a match |
| `DB2_CATALOG_PLAN_NAME` | 18 | `EXECTPLN` (CL8) | `GVBMRCT` | space-padded plan name (source comment `(MRCT)`) |
| `DB2_SQL_PLAN_NAME` | 10 | `EXECSPLN` (CL8) | `GVBMRSQ` | space-padded plan name used by `GVBMRSQ` (`MRSQ`) |
| `DB2_VSAM_PLAN_NAME` | 11 | `EXECVPLN` (CL8) | `GVBMRDV` | space-padded plan name used by the external `GVBMRDV` |
| `DB2_VSAM_DATE_FORMAT` | 22 | `EXEC_db2_df` (CL3) | `ISO` | one of `ISO`, `JIS`, `DB2`, `USA`, `EUR` (upper-cased with `OC ...,SPACES`) |
| `DISK_THREAD_LIMIT` | 06 | `EXECDISK` (CL4) | `9999` | numeric; must not be zero (`clc execdisk,zeroes`) |
| `DUMP_LT_AND_GENERATED_CODE` | 05 | `EXECSNAP` (CL1) | `N` | `Y`/`N`; `Y` SNAPs the logic table and generated code to `EXTRDUMP` |
| `EXECUTE_IN_PARENT_THREAD` | 03 | `EXECSNGL` (CL1) | `N` | `1` = run the first ES set on the mother TCB only; `A` = run all ES sets sequentially on the mother TCB; `N` = normal parallel sub-tasks |
| `FISCAL_DATE_DEFAULT` | 17 | `exec_fdate` (CL8) | blanks | 8-digit `ccyymmdd`, or `VDP` to take the fiscal date from VDP 0001 (`clc 0(3,r4),=cl3'VDP'`); errors `FISCALDATE_LEN_ERR`, `FISCALDATE_DEFAULT_NOT_NUM`, `FISCALDATE_VALUE_ERR` |
| `FISCAL_DATE_OVERRIDE` | 23 | linked `fiscal_date_entry` chain | none | `recid:ccyymmdd`; errors `FISCALDATE_CTRL_REC_ERR`, `FISCALDATE_NOT_NUM`, `FISCALDATE_CTRL_REC_DUP`, `FISCALDATE_LEN_ERR` |
| `INCLUDE_REF_TABLES_IN_SYSTEM_DUMP` | 27 | `EXEC_Dump_Ref` (CL1) | `Y` | `Y`/`N` (upper-cased); validated and echoed in the control report. No other statement in this repository reads `EXEC_Dump_Ref`, so the shipped source does not act on it |
| `IO_BUFFER_LEVEL` | 21 | `EXECMSDN` (CL3) | `004` | numeric, right-justified; passed to `DCBE MULTSDN` |
| `LOG_MESSAGE_LEVEL` | 28 | `EXEC_LOGLVL` (CL8) | `STANDARD` | `STANDARD` or `DEBUG`; `DEBUG` sets `WORKFLAG1` bit `MSGLVL_DEBUG` |
| `OPTIMIZE_PACKED_OUTPUT` | 24 | `exec_optpo` (CL1) | `Y` | `Y`/`N` |
| `PAGE_FIX_IO_BUFFERS` | 15 | `execpagf` (CL1) | `N` | `Y`/`N`; `Y` lets `GVBMRBS` and the extract writer page-fix their BSAM buffers. `EDITPARM` issues `TESTAUTH FCTN=1` and, if the job is not APF-authorized, forces the value back to `N` and logs `NO_PAGE_FIX` |
| `RECOVER_FROM_ABEND` | 26 | `exec_estae` (CL1) | `Y` | copied as-is; only `Y` makes each thread issue `ESTAEX ... PARAM=(R2),PURGE=HALT` for `TASKABND` (and load the overflow mask); `N` leaves abends to z/OS |
| `RUN_DATE` | 16 | `exec_rdate` (CL8) | blanks | 8-digit `ccyymmdd`, or `VDP`; also copied to `exec_fdate` when no fiscal date was given; errors `RUNDATE_LEN_ERR`, `RUNDATE_INVALID`, `RUNDATE_NOT_NUM` |
| `SOURCE_RECORD_LIMIT` | 08 | `EXECRLIM` (CL13) | `0000000000000` | numeric, right-justified into 13 digits (`ZEROES13`); 0 = no limit; checked per thread in the record loop |
| `TAPE_THREAD_LIMIT` | 07 | `EXECTAPE` (CL4) | `9999` | numeric |
| `TRACE` | 04 | `EXECTRAC` (CL1) | `N` | `Y`/`N`; `Y` enables the trace hooks in generated code and reads `EXTRTPRM` |
| `TREAT_MISSING_VIEW_OUTPUTS_AS_DUMMY` | 09 | `EXECDUMY` (CL1) | `N` | `Y`/`N`; `Y` builds a dummy `EXTFILE` for extract DDNAMEs missing from the TIOT instead of failing the open |
| `USE_ZIIP` | 12 | `EXECZIIP` (CL1) | `N` | `Y`/`N`, checked in `EDITPARM` (`ziip_parm_invalid`); `Y` requires `GVBMRZP` to be linked, otherwise `ZIIP_FEATURE_NOT_AVAILABLE` (511) and RC 8 |
| `VERIFY_CREATION_TIMESTAMP` | 29 | `EXEC_check_timestamp` (CL1) | `Y` | `Y`/`N`; `Y` compares the VDP 0001 timestamp with the logic table `GEN` timestamp (`VDP_XLT_TIMESTAMP_ERR` 101 on mismatch) |
| `ZIIP_THREAD_LIMIT` | 13 | `EXEC_SRBLIMIT` (CL4) | `9999` | numeric; maximum number of concurrently scheduled SRBs |
| `HASH_PACK` | 30 | `EXEC_HASHPACK` (CL1) | `N` | `Y`/`N`; pack the key before `CKSM` hashing |
| `HASH_MULT` | 31 | `EXEC_HASHMULT` (CL2) / `EXEC_HASHMULTB` (XL4) | `3` | numeric 1–10 (`if CHI,R0,GT,10,or,CHI,R0,lt,1` → `PARM_ERR`) |
| `DISPLAY_HASH` | 32 | `EXEC_DISPHASH` (CL1) | `N` | `Y`/`N`; `Y` opens the `Hnnnnnnn` hash statistics report (`Open_hash_report`) |
| `HASH_TABLE_LU` | 33 | `HASH_LU_ENT` chain | none | `(LF,LR,mult,PACK\|NOPACK)`; `LF` may be `*` for all logical files; sub-values separated by commas (`cli 0(r1),c','`) |
| `UTILITY` | 34 | `EXEC_DB2HPU` (CL1) | `N` | upper-cased with `OI ...,X'40'`; echoed in the control report. Access method 16 (`DB2HPU` from VDP 0200) — not this keyword — is what actually selects `GVBMRSU` |

Fields in `EXECDATA` that no keyword sets but that exist in the layout:
`EXECVERS` (`FL4'004'`, logic-table version accepted), `EXECNRD`/`EXECNWRT`
(read/write buffer counts, carried but not parsed from `EXTRPARM`), `EXECVSIZ`
(`001000`, VDP table size in K), `exec_bdate` (batch date, set from the system
date). The complete DSECT is reproduced in
[12-control-blocks.md](12-control-blocks.md).

Records whose first non-blank character is `*` are comments. After the file
is exhausted, `EDITPARM` re-validates the switches that were copied without
checking (`USE_ZIIP`, `ABEND_ON_CALCULATION_OVERFLOW`), converts the overflow
switch to the binary `ovflmask`, and downgrades `PAGE_FIX_IO_BUFFERS` when
the step is not APF-authorized. Every keyword and its final value is then
echoed into the control report (`EXTRRPT`) under the *parameters* heading,
which is why the table is kept in alphabetical order.

### 13.1.3 `TRACE` sub-parameters (`EXTRTPRM` / `REFRTPRM`)

When `TRACE=Y` the trace-parameter file is read (DCB `TPRMDCB`) with the same
keyword mechanism (`EX R15,TPRMCLC`) against `TraceParmtable`:

```asm
TraceParmtable dc 0H
         DC    CL35'VIEW        ',H'01'
         DC    CL35'FROMREC     ',H'02'
         DC    CL35'THRUREC     ',H'03'
         DC    CL35'FROMLTROW   ',H'04'
         DC    CL35'THRULTROW   ',H'05'
         DC    CL35'LTFUNC      ',H'06'
         DC    CL35'DDNAME      ',H'07'
         DC    CL35'VPOS        ',H'08'
         DC    CL35'VLEN        ',H'09'
         DC    CL35'VALUE       ',H'10'
         DC    CL35'DISPLAYSOURCE',H'11'   INCL EVENT RECS IN TRACE
         DC    CL35'LTROW        ',H'12'
         DC    CL35'REC          ',H'13'
         DC    CL35'COL          ',H'14'
         DC    CL35'FROMCOL      ',H'15'
         DC    CL35'THRUCOL      ',H'16'
         DC    XL4'FFFFFFFF'
```

Each `VIEW=` record starts a new `PARMTBL` entry (`MAC/GVBMR95C.mac`); the
other keywords fill its fields. `LTROW`, `REC` and `COL` are single-value
shorthands that set both the *from* and *thru* fields.

| Trace keyword | `PARMTBL` field(s) | Meaning |
|---|---|---|
| `VIEW` | `PARMVIEW` (FL4) | view number to trace (`0` = all views, *inferred from the compare in `MR95TRAC`*) |
| `FROMREC` / `THRUREC` / `REC` | `PARMFROM`, `PARMTHRU` (XL8) | event-record number range |
| `FROMLTROW` / `THRULTROW` / `LTROW` | `PARMROWF`, `PARMROWT` (`PARMrows`) | logic-table row range |
| `LTFUNC` | `PARMFUNC` (CL4) | function code filter, e.g. `DTE`, `CFEC` |
| `DDNAME` | `PARMDDN` (CL8) | only trace records read from this event DDNAME |
| `VPOS` / `VLEN` / `VALUE` | `PARMVOFF`, `PARMVLEN`, `PARMVALU` (CL16) | only trace records whose bytes at `VPOS` for `VLEN` equal `VALUE` |
| `DISPLAYSOURCE` | `PARMDUMP` (CL8) | also print the event record in the trace |
| `FROMCOL` / `THRUCOL` / `COL` | `PARMfcol`, `PARMtcol` (`PARMcols`) | extract-column range |

`PARMLTAB` and `PARMSGAB` exist in the `PARMTBL` layout for the abend row /
abend message values, but the current source keeps those values in the
work-area fields `abend_lt` and `abend_msg` instead (the `EXECDATA` comments
say *moved from PARMTBL*); no statement writes `PARMLTAB`/`PARMSGAB`. Errors
in this file raise `TRACE_PARM_ERR` (177). Entries are chained through `PARMNEXT`; each `NV` row's
`LTPARMTB` points at the first entry for its view so the generated
`BAS R10,MR95TRAC` hook can filter quickly (see
[06-generated-code.md](06-generated-code.md)).

Trace output goes to `EXTRTRAC` (`RECFM=FBA,LRECL=161`).

### 13.1.4 `RUNVIEWS`: restricting the run to a subset of views

`RUNVIEWS` is optional. `GVBMR96` scans the TIOT (`tioeddnm`) and only opens
the file if the DD is present. Records beginning with `*` are comments; each
other record carries one numeric view number, validated with the unsigned
numeric `TRT` table (`RUNVIEW_NOT_NUM`, `RUNVIEW_OUT_OF_RANGE`) and inserted
in ascending order into `runview_list` (`runv_count`, `runv_first`,
`runvent_view`/`runvent_next`). A view number that does not exist in the
logic table raises `RUNVIEW_NOT_EXIST`.

The list is applied in two places:

* PASS2 (`P2FUNNV` in `GVBMR95`): a view that is *not* in the list has its
  `NV` prologue disabled with `oi nvnop+1,x'f0'`, which turns the prologue's
  `NOP` into an unconditional `BC 15`, so the view's generated code is
  bypassed for every record.
* The control report (`runv_check`): only listed views are reported.

### 13.1.5 Environment variables (`EXTRENVV` / `REFRENVV`)

`EXTRENVV` records are `NAME=VALUE` pairs loaded by `GVBMR96` and used by
`GVBUR33` to substitute `&NAME` / `%NAME%` style symbolics in VDP text (data
set names, DDNAMEs, SQL). See `GVBUR33` in [11-utilities.md](11-utilities.md).

## 13.2 Format phase parameters (`MR88PARM`)

`READPARM` in `GVBMR87` reads `MR88PARM` until EOF, skipping `*` records, and
matches the active keyword table:

```asm
parmk09v dc    C'ABEND_ON_MESSAGE_NBR         '
parmk06v dc    C'FISCAL_DATE_DEFAULT          '
parmk07v dc    C'FISCAL_DATE_OVERRIDE         '
parmk05v dc    C'RUN_DATE                     '
parmk03v dc    C'PROCESS_HEADER_RECORDS       '
parmk04v dc    C'SORT_EXTRACT_FILE            '
```

| Keyword | Stored in | Behaviour |
|---|---|---|
| `SORT_EXTRACT_FILE=Y\|N` | `EXTROPT` | `Y`: `GVBMR87` invokes SORT (`SORTINIT`) and receives records in `SORTE35`; `N`: `MR88HXE` is read directly with BSAM as already-sorted input |
| `PROCESS_HEADER_RECORDS=Y\|N` | – | parsed but ignored (the `MVC NOHDROPT,0(R4)` is commented out); `NOHDROPT` is derived from `EXTROPT` at EOF |
| `RUN_DATE=ccyymmdd` | `svrundt` | overrides `vdp0001_run_date` |
| `FISCAL_DATE_DEFAULT=ccyymmdd` | fiscal default | duplicate keyword rejected |
| `FISCAL_DATE_OVERRIDE=recid:ccyymmdd` | fiscal chain | per control-record fiscal date |
| `ABEND_ON_MESSAGE_NBR=nnn` | `abend_msg` | `RTNERROR` executes `DC XL4'FFFFFFFF'` (an invalid op-code, hence S0C1) when that message is raised |

Details and message numbers are in
[10-format-phase-gvbmr87-gvbmr88.md](10-format-phase-gvbmr87-gvbmr88.md).

## 13.3 DDNAMEs

### 13.3.1 Extract phase (`GVBMR95E`) and reference phase (`GVBMR95R`)

The DCBs are declared once with the `EXTR*` names. When the alias is
`GVBMR95R`, `GVBMR96` overwrites `DCBDDNAM` in the copied DCB with the
`REFR*` name before `OPEN`:

```asm
mvc  rpt_dd95C,=cl8'EXTRRPT'
if clc,namepgm,eq,=cl8'GVBMR95R'
  mvc  rpt_dd95C,=cl8'REFRRPT'
  mvc  dcbddnam,=cl8'REFRRPT'
endif
```

| Extract DDNAME | Reference DDNAME | DCB (source) | Direction | Attributes in source | Contents |
|---|---|---|---|---|---|
| `EXTRPARM` | `REFRPARM` | `PARMDCB` (`GVBMR96`) | in | `MACRF=(GL)`, `RMODE31=BUFF` | execution parameters (13.1.2) |
| `EXTRTPRM` | `REFRTPRM` | `TPRMDCB` | in | `MACRF=(GL)` | trace parameters (13.1.3); only opened when `TRACE=Y` |
| `EXTRENVV` | `REFRENVV` | `ENVVDCB` | in | `MACRF=(GL)` | environment variables for `GVBUR33` |
| `MR95VDP` | `MR95VDP` | `VDPDCB` | in | `MACRF=(GL)` | View Definition Parameters (binary, RDW-prefixed records) |
| `EXTRLTBL` | `REFRLTBL` | `LTBLdcb` | in | `MACRF=(GL)` | logic table (XLT for extract, JLT for reference) |
| `EXTRREH` | `REFRREH` | `HDRDCB` | in | `MACRF=(GL)`, `EODAD=FILLEOF` | reference-table header records (REH); one record per reference table produced by the reference phase |
| `GREFnnn` (from REH) | – | `GVBUR20` | in | sequential | reference-table data (RED) loaded into `LKUPBUFR` |
| `RUNVIEWS` | `RUNVIEWS` | `runvdcb` | in | `MACRF=(GL)` | optional view subset (13.1.4); opened only if present in the TIOT |
| VDP 0200 DDNAME | same | `EVNTFILE` model | in | `MACRF=(R)`, `EODAD=0` (exit installed at run time) | event files; DDNAME taken from the physical-file record; dynamically allocated by `GVBUR35` when the VDP supplies a data set name and the DD is absent |
| `EXTRnnn` (`LTWRFILE` from the `WR` row) | `REFRnnn`/table DDNAMEs | `EXTRFILE` model (`DDNAME=EXTR`) | out | `MACRF=(W)`, `RECFM=VB,LRECL=8192`, `BLOCKTOKENSIZE=LARGE` | extract files (one per `WR` target); may be pipes/tokens or dummies |
| `EXTRRPT` | `REFRRPT` | `CTRLFILE` | out | `RECFM=VB,LRECL=164` | control report: parameters, views, files, counts, timings |
| `EXTRLOG` | `REFRLOG` | `logfile` (`GVBMR95`) | out | `RECFM=VB,LRECL=164` | `GVBMSG LOG` messages |
| `EXTRTRAC` | `REFRTRAC` | `TRACfile` | out | `RECFM=FBA,LRECL=161` | logic-table execution trace |
| `EXTRDUMP` | `REFRDUMP` | `SNAPDCB` | out | `MACRF=(W)` | `SNAP` output for `DUMP_LT_AND_GENERATED_CODE=Y` and abend recovery |
| `Hnnnnnnn` | same | `HASHFILE` | out | `RECFM=VB,LRECL=164` | hash statistics per lookup (`DISPLAY_HASH=Y`); `nnnnnnn` is built from the lookup record ID |
| `SORT` | `SORT` | `SORTFILE` (`GVBMR95`) | out | `RECFM=FB,LRECL=80` | sort control statements generated for the extract files (see 3.10 in [03-gvbmr95-extract-engine.md](03-gvbmr95-extract-engine.md)) |

The `EXTRREH`/`REFRREH` distinction matters: the reference phase *writes*
REH/RED files (through `WR` rows whose targets are the reference DDNAMEs), the
extract phase *reads* them through `EXTRREH`/`GREFnnn` to build lookup
buffers ([07-lookups-and-reference-data.md](07-lookups-and-reference-data.md)).

### 13.3.2 Format phase (`GVBMR88`, opened by `GVBMR87`)

| DDNAME | DCB | Direction | Attributes in source | Contents |
|---|---|---|---|---|
| `MR88PARM` | `PARMDCB` | in | `MACRF=(GL)` | format-phase parameters (13.2) |
| `MR88VDP` | `VIEWFILE` | in | `MACRF=(GL)`, `EODAD=VDPEOF` | VDP (only record types 0001, 0002, 0050, 0400, 0650, 1000… are used) |
| `MR88HXE` | `EXTRFILE` | in | `MACRF=(R)`, `RECFM=VB`, `BLOCKTOKENSIZE=LARGE` | extract file (sorted, or sorted in-flight through SORT E15/E35) |
| `REFRRTH` | `HDRFILE` | in | `MACRF=(GL)`, `EODAD=FILLEOF` | reference-table headers for format-phase lookups |
| `MR88RTD` | `LKUPFILE` | in | `MACRF=(GL)` | reference-table data |
| `SYSIN` | `SYSIN` | in | `MACRF=(GL)`, `EODAD=SYSINEOF` | user sort control statements copied into the generated SORT deck |
| `MR88DATA` | `DATAFILE` | out | `MACRF=(PL)`, `RECFM=FB`, `BUFNO=20` | fixed-format data output (view output media `FILE`) |
| `MR88PRNT` | `PRNTFILE` | out | `RECFM=VBA,LRECL=137,BLKSIZE=13700` | printed reports |
| `MR88RPT` | `CTRLFILE` | out | `RECFM=VB,LRECL=164` | control report |
| `MR88LOG` | `LOGFILE` | out | `RECFM=VB,LRECL=164` | `GVBMSG LOG` messages |
| `SNAPDATA` | `SNAPDCB` | out | `RECFM=VBA,LRECL=125,BLKSIZE=1632` | `SNAP` dumps |
| per-view DDNAMEs | `VWDCBADR` | out | from VDP 1000 | view output files (`FILE`, CSV, XML, online) |

### 13.3.3 Utilities

`GVBUT99` (user abend) takes its abend code from the `EXEC PARM`.
`GVBUR35` needs no DDNAMEs (it builds SVC 99 text units for the caller).
`GVBUR39` scans the joblib/steplib and link-list for `GENWRITE` load modules
via `BLDL`. See [11-utilities.md](11-utilities.md).

## 13.4 Return codes and abends

### 13.4.1 `GVBMR95E` / `GVBMR95R`

`GVBMR95` keeps the highest completion code seen in `overall_return_code`:

```asm
llgt r15,0(0,r1)      get the ecb contents
nilf r15,x'00ffffff'  clean out the top byte
if cl,r15,gt,overall_return_code  is this largest so far?
  st r15,overall_return_code save so we know we're failing
endif
```

and at `Return` either returns it in R15 or, if a sub-task failed
(`thread_fail`), re-issues the failure as an abend of the main task
(`ABEND (1),,,USER` for a user code, `ABEND (1),,,SYSTEM` after shifting the
system code right by 12).

| RC | Origin |
|---:|---|
| 0 | normal completion |
| 1 | `GVBMR96` found an empty logic table: `EMPTY_REF_LOGIC_TBL` (809) or `EMPTY_LOGIC_TBL` (`lghi R15,1` in `RTNERR05`); the run ends without extracting anything |
| 4 | warning path (`if Cij,R2,lt,4 / la r2,4`), e.g. `PIPE_WARN` (811) when pipes are used in single-thread mode, `GVBURALI` reporting no alias |
| 8 | `Return_err`: any `RTNERROR` in `GVBMR96` other than the empty-logic-table cases, parameter errors, open failures, `ZIIP_FEATURE_NOT_AVAILABLE`, driver unavailable |
| 12 | `EXEC_PGM_ERR`: the load module was entered under its base name `GVBMR95` instead of an alias |
| 16 | pause-element/ESTAE infrastructure failure (`PE_error`), abort after abend recovery |
| U0999 | `ABEND_ON_ERROR_CONDITION=Y` and an error occurred (`abend 999` in `Return_err`) |
| Unnn / Sxxx | a thread abended: `TASKABND` logs `THREAD_ABEND` (9) and `FAILING_ROW` (602), and the main task re-abends with the same code |

Lookup exits use their own convention (0 = found, 4 = not found, 8 = skip
event record, 12 = disable view, 16+ = abort the run); read and write exits
are described in [09-io-handlers.md](09-io-handlers.md) and
[03-gvbmr95-extract-engine.md](03-gvbmr95-extract-engine.md).

### 13.4.2 `GVBMR88`

The program prologue documents only `0 - SUCCESSFUL` and `8 - ERROR`.

| RC | Origin |
|---:|---|
| 0 | normal completion |
| 8 | `RTNERRORX`: files are closed (`closfile`) and `LHI R15,8`; the message was written to `MR88LOG` first |
| 16 | returned to SORT (not to z/OS) when the error occurs while `GVBMR88` is running as the E35 exit (`SORTRSA` non-zero): *RC=16 (DON'T CALL AGAIN)*; SORT then terminates and the step ends with SORT's completion code |
| S0C1 | `ABEND_ON_MESSAGE_NBR` matched: `DC XL4'FFFFFFFF'` is executed |

### 13.4.3 `GVBUTMSG`

The message builder returns its own codes in R15 (`MAC/GVBUTEQU.mac` /
`ASM/GVBUTMSG.asm`):

```asm
MBSNOSUB EQU  4        substitution requested but no value supplied
MBSBUFOF EQU  8        output buffer overflowed
MBSNOFND EQU  12       message number not in the table
MBSNOFOF EQU  16       not found *and* buffer overflow
MBSNOBUF EQU  20       no buffer supplied for FORMAT
```

## 13.5 The message subsystem

```
   program                           MAC/GVBUTEQU.mac           MAC/GVBMSGGE.mac
   ───────                           ────────────────           ────────────────
   GVBMSG LOG,MSGNO=IO_ERROR, ──►    IO_ERROR EQU 8    ──►      GVBMSGDF 008,'&&1 - I/O error ...',TYPE=U
          SUBNO=2,SUB1=..,SUB2=..           │                            │
          │                                 │                            │  expands (GVBMSGDF) into
          ▼                                 │                            ▼
   GENMSG parameter list (MF=L/E)           │                   GVBUTMUE CSECT: directory + elements
          │                                 │                            ▲
          ▼                                 │                            │  L R11,=V(GVBUTMUE)
   L 15,=V(GVBUTMSG) / BASR 14,15  ─────────┴────────────────────────────┘
          │
          ├── directory search by number (groups of messages, first-ID + offset)
          ├── substitute &&1 … &&8 with SUBn pointers/lengths
          └── LOG → PUT to the log DCB   |  WTO → WTO   |  FORMAT → return text in caller buffer
```

### 13.5.1 The `GVBMSG` macro (`MAC/GVBMSG.mac`)

```asm
&LABEL    GVBMSG &TYPE,&GENENV=,&MSGNO=0,&MSGPFX=GVB,&SUBNO=0,&SUB1=,&S+
               UB2=,&SUB3=,&SUB4=,&SUB5=,&SUB6=,&SUB7=,&SUB8=,&MF=,&MSG+
               ...
```

* `&TYPE` is `LOG`, `WTO` or `FORMAT`.
* `GENENV=` points at the `GENENV` block (which carries the log DCB address
  and the run/thread identification printed in every message).
* `MSGNO=` is the message number, normally one of the `GVBUTEQU` symbols;
  `MSGPFX=` is the three-letter prefix (`GVB` by default).
* `SUBNO=n` and `SUB1=`…`SUB8=` supply substitution values. Each `SUBn` is
  either `(addr,len)` or a field name (length taken from the field).
* `MF=L` builds a static `GENMSG` list; `MF=(E,area)` fills an existing list
  and calls `GVBUTMSG`. The programs load the routine address with
  `L 15,=V(GVBUTMSG)` and `BASR 14,15`.

The parameter list DSECT (`GENMSG`):

```asm
GENMSG      DSECT
MSGTYPE     DS AL4            C'L' log, C'W' WTO, C'F' format
MSGGENV     DS A              -> GENENV
MSGDCBA     DS A              -> log DCB (LOG)
MSGPFX      DS A              -> 3-byte prefix
MSGNUM      DS AL4            message number
MSGBUFFA    DS A              -> output buffer (FORMAT)
MSGBUFFL    DS AL4            buffer length
MSG#SUB     DS AL4            number of substitutions
MSGS1PTR    DS A              -> substitution 1
MSGS1LEN    DS AL4            length of substitution 1
...                           through MSGS8PTR / MSGS8LEN
```

### 13.5.2 Message catalogue (`GVBMSGDF` → `GVBMSGGE` → `GVBUTMUE`)

`MAC/GVBMSGGE.mac` is the single source of message text. It is a list of
`GVBMSGDF` invocations assembled into the `GVBUTMUE` CSECT ("Mixed-case
message table" in `TABLE/PGM.csv`):

```asm
GVBMSGDF GVB,TYPE=START
GVBMSGDF 000,'&&1 - Message not known ',TYPE=S
GVBMSGDF 001,'&&1 - IDENTIFY EP=MR95THRD macro failed. RC=&&2',TYPE=S
GVBMSGDF 002,'&&1 - &&2 threads started',TYPE=I
GVBMSGDF 003,'&&1 - ATTACH macro failed. RC=&&2. &&3',TYPE=S
GVBMSGDF 006,'&&1 - Source file header record version wrong',TYPE=U
...
GVBMSGDF GVB,TYPE=END
```

`GVBMSGDF &ID,&MSG,&TYPE=` (`MAC/GVBMSGDF.mac`) validates `&TYPE` against
`I W E S C U N`, and emits for each message:

```asm
MSG&ID   DC    AL4(&WORK)          message number
         DC    AL1(EL&ID)          element length
         DC    AL1(ML&ID)          text length
msgt&id  DC    C'&outID&TYPE '     e.g. 'GVB0008U '
         DC    C&MSG               text with &&1..&&8 placeholders
ML&ID    EQU   *-MSGt&id
EL&ID    EQU   *-MSG&ID
```

`TYPE=START` opens the table and `TYPE=END` closes it and generates the
*directory*: messages are grouped, and each directory entry holds the first
message ID of the group and the offset of the group, so `GVBUTMSG` can jump
to the right group and then scan linearly. `TYPE=DSECT` produces the mapping
DSECT for consumers.

Distribution of the 340 active entries by type letter:

| Type | Count | Convention (from message text and usage) |
|---|---:|---|
| `U` | 215 | unrecoverable: the run stops (RC 8 or abend) |
| `S` | 101 | severe: error in a view/file/thread, normally also terminal |
| `I` | 16 | informational (`threads started`, timings, counts) |
| `W` | 8 | warning (e.g. `PIPE_WARN`) |

`E`, `C` and `N` are accepted by `GVBMSGDF` but are not used by any active
entry. The type letter is embedded in the text (`GVBnnnT`) and does **not**
drive `GVBUTMSG`'s return code; the calling program decides how to react by
message number (`ERRMSG#`, `RTNERROR`).

### 13.5.3 Symbolic message numbers (`MAC/GVBUTEQU.mac`)

Programs never code message numbers as literals; they use the ~338 `EQU`
symbols in `GVBUTEQU`, which are grouped by subsystem:

| Range | Area | Examples |
|---|---|---|
| 000–099 | `GVBMR95` thread/task infrastructure | `IDENTIFY_FAIL` 1, `NUM_THREADS` 2, `ATTACH_FAIL` 3, `IO_ERROR` 8, `THREAD_ABEND` 9, `PARM_ERR` 33, `DYNALLOC_FAIL` 44, `ENV_VAR_ERR` 51, `OPEN_LTBL_FAIL` 52, `OPEN_VDP_FAIL` 53 |
| 100–199 | `GVBMR96` VDP / logic table / lookups / drivers | `VDP_XLT_TIMESTAMP_ERR` 101, `OPEN_EXTR_FAIL` 104, `OPEN_REH_FAIL` 106, `OPEN_REF_FAIL` 107, `LKUPBUFR_LF_NOT_FOUND` 120, `REF_READ_ERR` 124, `IO_DRIVER_UNAVAILABLE` 141, `REH_COUNT_ERR` 164, `RUNVIEW_NOT_EXIST` 167, `DB2_SQL_UNAVAILABLE` 197, `DB2_HPU_UNAVAILABLE` 198 |
| 200–299 | Adabas / exits / access methods | `ADABAS_UNAVAILABLE` 212 |
| 300–399 | DB2 (`GVBMRSQ`, `GVBMRSU`, `GVBMRHPU`) | `DB2_VSAM_OPEN_FAIL` 300, `SQL_PREPARE_FAIL` 309 |
| 400–499 | format phase (`GVBMR87`/`GVBMR88`) | `MSG#416`…`MSG#454`, `MSG#499` |
| 600–699 | thread end / recovery | `THREAD_END` 601, `FAILING_ROW` 602 |
| 800–899 | VDP structure / reference phase | `MISSING_VDP0801` 801, `EMPTY_REF_LOGIC_TBL` 809, `PIPE_WARN` 811 |
| 900+ | internal consistency | `CK_BUFFER_OVERFLOW` 900 |

Because every message begins with `&&1`, the first substitution is by
convention the program/thread identifier (e.g. `GVBMR95E`, `GVBMR96`,
`Thread nnn`), so log lines can be attributed even when several sub-tasks
write to the same `EXTRLOG`.

### 13.5.4 Where messages go

| Program | Log DDNAME | WTO | Also |
|---|---|---|---|
| `GVBMR95`, `GVBMR96`, I/O drivers | `EXTRLOG` / `REFRLOG` | for abend-recovery and pause-element failures | control report `EXTRRPT`/`REFRRPT`; trace `EXTRTRAC` |
| `GVBMR87`, `GVBMR88` | `MR88LOG` (`MSGTYPE=C'L'`) | – | control report `MR88RPT` |
| `GVBMRHPU` (`INZEXIT`) | HPU's own log via the exit parameter list | – | – |
| `GVBUT99` | – | `WTO` of the abend text | `ABEND ...,DUMP` |

## 13.6 Abend controls summarised

```
  parameter                          field           effect
  ─────────────────────────────────  ──────────────  ─────────────────────────────────────────────
  ABEND_ON_ERROR_CONDITION=Y         exec_uabend     Return_err → ABEND 999 instead of RC 8
  ABEND_ON_MESSAGE_NBR=nnnnn         abend_msg       ERRMSG#/RTNERROR execute X'FFFFFFFF' (S0C1) when message nnnnn is issued
  ABEND_ON_LOGIC_TABLE_ROW_NBR=nnn   abend_lt        MR95TRAC (TRACABND) executes X'FFFFFFFF' when traced row nnn runs
  ABEND_ON_CALCULATION_OVERFLOW=Y    ovflmask        SPM sets fixed-point/decimal overflow mask → S0C8/S0CA on overflow
  RECOVER_FROM_ABEND=N               exec_estae      no ESTAEX in threads → raw system abend, no THREAD_ABEND/FAILING_ROW
  INCLUDE_REF_TABLES_IN_SYSTEM_DUMP  EXEC_Dump_Ref   parsed and echoed only (no consumer in this source)
  DUMP_LT_AND_GENERATED_CODE=Y       EXECSNAP        SNAP logic table + generated code to EXTRDUMP at start
  (MR88) ABEND_ON_MESSAGE_NBR=nnn    abend_msg       RTNERROR executes X'FFFFFFFF' → S0C1
```

The row abend lives inside the trace subroutine, so it needs `TRACE=Y` and at
least one `EXTRTPRM` entry covering the row; the message abend works
unconditionally. The dump produced by either is the primary debugging aid
for generated-code problems, since `TASKABND` maps the failing PSW back to a
logic-table row (`scan_lt`) and reports it as `FAILING_ROW`.
