# 10. Format Phase: `GVBMR87` and `GVBMR88`

The format (report) phase consumes the extract records written by the extract
phase (`GVBMR95E`), sorts them (either externally or by driving DFSORT/SyncSort
from inside the program), and produces the final outputs: hardcopy reports,
online/VSAM records, fixed-format files, CSV, and XML. It is implemented by two
modules:

| Module | Role | AMODE/RMODE | Return |
|---|---|---|---|
| `GVBMR88` | Main report program: reads the (sorted) extract file, detects sort-key breaks, accumulates totals in extended decimal floating point, formats columns and drives all output destinations | `AMODE 31`, `RMODE 31` | 0 ok, 4 warning-only (`MSG#414`), 8 error or column overflow |
| `GVBMR87` | Initialization subroutine called by `GVBMR88`: parses `MR88PARM`, loads the VDP, builds the `VIEWREC`/`SORTKEY`/`COLDEFN`/`CALCTBL` control blocks, allocates work areas, opens files, loads reference (title lookup) tables, and owns the sort E15/E35 callbacks | `AMODE 31`, `RMODE 31` | 0 ok, 8 error (message number in R14) |

All statements below are drawn from `ASM/GVBMR87.asm`, `ASM/GVBMR88.asm`,
`MAC/GVBMR88C.mac` (control-block DSECTs and equates), `MAC/GVBMR88W.mac`
(work area) and `MAC/DL96AREA.mac`.

## 10.1 Where the format phase sits

```text
 GVBMR95E (extract phase)
     |  EXTRnnn  : header/control records + detail extract records
     v
 +----------------------------------------------------------+
 |  GVBMR88                                                 |
 |    GETMAIN WORKAREA, save FP regs, set program mask       |
 |    CALL GVBMR87 ------------------------------------+     |
 |                                                     v     |
 |            GVBMR87: READPARM -> VDPLOAD -> ALLOCDYN       |
 |                     -> FILLLKUP -> OPENOUT -> OPENSTD     |
 |                     (SORT_EXTRACT_FILE=Y: LINK EP=SORT,   |
 |                      records come back via SORTE35)       |
 |                                                     |     |
 |    READLOOP  <--------------------------------------+     |
 |      |-- control record  (CTLREC)  -> sum counts          |
 |      |-- header  record  (HDRREC)  -> RPTBREAK / new view |
 |      |-- extract record  (EXTREC)  -> EXTPROC:            |
 |             KEYBREAK / CALCCOLM / EXCPCHK / SUM /         |
 |             COLBUILD -> GVBDL96 -> VSAMWRT | COMFILE      |
 |    READEOF -> RPTBREAK (grand totals) -> CTRLRPT -> CLOSE |
 +----------------------------------------------------------+
     |            |             |           |          |
  MR88PRNT     MR88DATA /    online      MR88RPT    MR88LOG
  (report)     view DDs      (VSAM via   (control   (messages)
               file/CSV/XML  GVBTP90)    report)
```

## 10.2 `GVBMR88` entry, work area and registers

`GVBMR88` obtains its work area with `GETMAIN R`, zeroes it with `MVCL`, and
chains the new save area to the caller's (`savprev` / `prevsa.savnext`):

```asm
         LHI   R0,WORKLEN+l'workeyeb
         GETMAIN R,LV=(0)
         MVC   0(l'workeyeb,R1),WORKEYEB
         LA    R13,l'workeyeb(,R1)
         USING WORKAREA,R13
         LR    R0,R13
         LHI   R1,WORKLEN
         SR    R14,R14
         SR    R15,R15
         MVCL  R0,R14
         ST    R10,savprev
         ST    R13,prevsa.savnext
```

Two details of the prologue are important for the numeric behaviour of the
whole phase:

1. **Extended DFP registers are part of the calling contract.** The source
   notes that `FP8/FP10` always holds a DFP zero and `FP9/FP11` holds the
   quantum for 3 or 8 decimal places, loaded by `GVBMR87`; `GVBMR88` saves
   `fp8`, `fp9`, `fp12`, `fp13` into `fp_reg_savearea` with `GVBSTX` and
   restores them at `returne`. All accumulators are 16-byte extended DFP
   (`AccumDFPl`), always handled as register pairs (0/2, 1/3, 4/6, 8/10 ...).
2. **Decimal-overflow program mask is turned off** before SORT is invoked:

```asm
         ipm   r14
         st    r14,wksavmsk
         nilh  r14,b'1111101111111111'   turn off decimal overflow
         spm   r14
```

   so a packed overflow sets the condition code instead of raising an `0CA`
   abend; the mask is restored from `wksavmsk` on return.

`GVBMR87` is then called as an ordinary subroutine; any nonzero R15 is a
message number and goes straight to `RTNERROR`:

```asm
         L     R15,GVBMR87A
         BASR  R14,R15
         LTR   R14,R15
         JNZ   RTNERROR
         MVC   PRNTSUBR,VSAMWRTA      print routine = VSAMWRT
         MVC   READSUBR,MERGEADR      read routine  = MERGEREC
```

Register conventions inside `GVBMR88` (from the program header):

| Register | Use |
|---|---|
| R13 | `WORKAREA` base and register save area |
| R11 | program base |
| R8 | current `VIEWREC` (report request) |
| R7 | current extract record (`EXTREC` / `HDRREC` / `CTLREC`) |
| R6 | current `SORTKEY`, `COLDEFN`, or `LKUPBUFR` prefix |
| R5 | loop counter / condition code / current reference record |
| R10, R9 | subroutine return registers (`BRAS R10,...`, `BRAS R9,...`) |
| R1 | parameter list |
| R15 | temporary and return code |

## 10.3 `GVBMR87` initialization sequence

`GVBMR87` runs the following subroutines in this exact order:

```asm
         brasl R10,PGMINIT        work area, time, LE, message file
         brasl R10,HEADINGS       control-report headings
         brasl R10,READPARM       MR88PARM keyword file
         brasl R10,PRINTRT1       control report part 1
         brasl R10,VDPLOAD        load MR88VDP into control blocks
         BRAS  R10,ALLOCDYN       dynamic work areas per view
         BRAS  R10,FILLLKUP       load title lookup (reference) tables
         brasl R10,PRINTRT2       control report part 2
         BRAS  R10,OPENOUT        open view output DCBs
         brasl R10,LERUNTIM       Language Environment for exits
         brasl R10,OPENSTD        open extract input (or start SORT)
```

`GVBMR87` register conventions: R13 = save area/work area (shared with
`GVBMR88`), R11 program base, R8 `VIEWREC`, R7 current column group, R6 current
sort-key/title group or calculation or `LKUPBUFR`, R5 current VDP record.

### 10.3.1 DDNAMEs owned by the format phase

The model DCBs are assembled in `GVBMR87` (`DCBAREA`) and copied into the work
area so the module stays reentrant:

| DDNAME | DCB label | MACRF / RECFM | Purpose |
|---|---|---|---|
| `MR88PARM` | `PARMDCB` | `GL`, `DCBE RMODE31=BUFF,EODAD=PARMEOF` | keyword parameters |
| `MR88VDP` | `VIEWFILE` | `GL`, `EODAD=VDPEOF` | view definition parameters |
| `MR88HXE` | `EXTRFILE` | `R`, `RECFM=VB`, `BLOCKTOKENSIZE=LARGE` | extract input (BSAM `READ`) |
| `MR88DATA` | `DATAFILE` | `PL`, `RECFM=FB`, `BUFNO=20` | model for view output files |
| `MR88PRNT` | `PRNTFILE` | `PM`, `RECFM=VBA,LRECL=137` | hardcopy report |
| `MR88RPT` | `CTRLFILE` | `PM`, `RECFM=VB,LRECL=164` | control report |
| `MR88LOG` | `LOGFILE` | `PM`, `RECFM=VB,LRECL=164` | messages (`GVBUTMSG`) |
| `REFRRTH` | `HDRFILE` | `GL`, `EODAD=FILLEOF` | reference table header (`TBLHEADR`) records |
| `MR88RTD` | `LKUPFILE` | `GL` | reference table data (renamed per header) |
| `SYSIN` | `SYSIN` | `GL`, `EODAD=SYSINEOF` | SORT control statements (only read when `SORT_EXTRACT_FILE=Y`) |
| `SNAPDATA` | `SNAPDCB` | `W`, `RECFM=VBA,LRECL=125` | diagnostic `SNAP` (open is commented out) |

Per-view output DDNAMEs come from the VDP (`VWDDNAME`); the DCB is cloned from
`DATAFILE`. An `RDJFCB` failure sets `VWNODD` (`MSG#405` DD statement missing).

### 10.3.2 `READPARM`: the `MR88PARM` file

`READPARM` sets defaults from `TIME BIN` / `TIME DEC`, opens `MR88PARM`, and
reads `keyword=value` records until EOF, skipping records beginning with `*`.
The active keyword table is:

```asm
parmk09v dc    C'ABEND_ON_MESSAGE_NBR         '
parmk06v dc    C'FISCAL_DATE_DEFAULT          '
parmk07v dc    C'FISCAL_DATE_OVERRIDE         '
parmk05v dc    C'RUN_DATE                     '
parmk03v dc    C'PROCESS_HEADER_RECORDS       '
parmk04v dc    C'SORT_EXTRACT_FILE            '
```

| Keyword | Effect | Errors |
|---|---|---|
| `SORT_EXTRACT_FILE=Y\|N` | `Y`: `GVBMR87` calls SORT itself (`SORTINIT`); extract records arrive through `SORTE35`. `N`: extract input is assumed already sorted and is read directly with BSAM. Stored in `EXTROPT`. | `MSG#433` |
| `PROCESS_HEADER_RECORDS=Y\|N` | Accepted but ignored: the `MVC NOHDROPT,0(R4)` is commented out ("Parm not used - SORT_EXTRACT_FILE parm now dictates this"). At EOF of the parm file `EXTROPT=Y` sets `NOHDROPT=N` (header records expected in the sorted stream) and `EXTROPT=N` sets `NOHDROPT=Y` (pre-sorted input, no headers). | – |
| `RUN_DATE=ccyymmdd` | Overrides `vdp0001_run_date` in `svrundt`; must be numeric and 8 long. | `MSG#447`, `MSG#448` |
| `FISCAL_DATE_DEFAULT=ccyymmdd` | Single default fiscal date; duplicate keyword is rejected. | `MSG#443`, `MSG#444` |
| `FISCAL_DATE_OVERRIDE=recid:ccyymmdd` | Builds a linked table of per-control-record fiscal dates. | `MSG#441`, `MSG#442`, `MSG#445`, `MSG#446` |
| `ABEND_ON_MESSAGE_NBR=nnn` | Saved in `abend_msg`; `RTNERROR` executes `DC XL4'FFFFFFFF'` (forced abend) when that message number is raised. | `MSG#417` |

Any other keyword is `MSG#416`. Older keywords for calculation-overflow and
zero-divide handling remain in the source only as comments (`MSG#435`,
`MSG#440` are still defined in `GVBUTEQU`).

### 10.3.3 `VDPLOAD`: VDP records used by the format phase

`VDPLOAD` opens `MR88VDP` (`MSG#418` on failure) and processes only the record
types the format phase needs (source comment `rtc20047 ignore VDP record types
not required by MR88`):

| VDP type | Action in `GVBMR87` |
|---:|---|
| 0001 | Save run number/date/description; `VDP0001_MAX_DECIMAL_PLACES=3` sets `LRGNUM=Y` (large-number mode); `GETMAIN` the LR field table (`FLDDEFTB`, `FDENTLEN` × `VDP0001_LR_FIELD_COUNT`); `STORAGE OBTAIN` a page-aligned VDP copy area (`VDPBEGIN`, `vdp_seg_len`) |
| 0002 | `GETMAIN` the column definition table (`COLDEFTB`, `CDENTLEN` × `vdp0002_totaldt_columns`, minimum 1) |
| 0050 | control record |
| 0400 | LR field definitions -> `FLDDEFN` entries |
| 0650 | lookup path / reference metadata used by `FILLLKUP` |
| 1000 | one `VIEWREC` per view (chained from `VWCHAIN`), only for format-phase views (`check_view_list`) |
| 1210 | exception condition stack -> `VWEXCOND` / `VWEXCNT` |
| 1300 / 1400 | report title / footer lines (`VWRTADDR`, `VWRFADDR`, `SK1300*`, `SK1400*`) |
| 1600 | summary output file attributes |
| 2000 | column records -> `COLDEFN` |
| 2210 | column calculation stack -> `CALCTBL` chain (see 10.7) |
| 2300 | sort key attributes -> `SORTKEY` |

Table overflows are reported with `MSG#408` (fields), `MSG#413` (columns),
`MSG#419` (calculations), `MSG#410`/`MSG#423` (sort keys); an empty VDP is
`MSG#425`. Validation of limits: sort key length must be ≤256 (`MSG#452`),
≤150 for printed reports (`MSG#453`), report width ≤256 (`MSG#454`).

### 10.3.4 `ALLOCDYN`, `FILLLKUP`, `OPENOUT`

* `ALLOCDYN` allocates, per view: sort-key save areas (`SVSORTKY`), the
  normalized column array (`EXTCOLA`), pre-calculation save (`EXTPCALC`),
  last-value (`EXTPREVA`) and first-value (`EXTFRSTA`) areas, break-level
  subtotal sets (`SUBTOTAD`, `VWSETLEN` bytes per level), and title/footer
  work areas. Column offsets are published in `CLCOFFTB` (indexed by column
  number) and the calculated-column index list `CLCCOLTB`.
* `FILLLKUP` reads `REFRRTH` headers and `MR88RTD` data into memory-resident
  `LKUPBUFR`/`LKUPTBL` structures and builds binary-search paths for sort-key
  title lookups (`MSG#426`–`MSG#430`, `MSG#438`). See
  [07-lookups-and-reference-data.md](07-lookups-and-reference-data.md).
* `OPENOUT` opens each view's output DCB (`MSG#403`), checks `LRECL` against
  `VWOUTLEN` (`MSG#404`), loads format exits named in `VWFMTPGM`
  (`MSG#407`), and starts Language Environment when an exit needs it
  (`LERUNTIM`, `MSG#432`/`MSG#437`).

### 10.3.5 `OPENSTD`: extract input and SORT integration

`OPENSTD` chooses the input path from `EXTROPT`:

```text
EXTROPT = 'N'  -> OPEN MR88HXE (BSAM), READSUBR uses the DCB READ/CHECK
                  addresses, extract DECBs + page-aligned buffers allocated
                  dynamically. Empty file -> EXTREOF='Y'.
EXTROPT = 'Y'  -> SORTINIT: build SORT control statements, LINK EP=SORT,
                  E35 = SORTE35, fixed 8K block, 128 buffers.
```

The sort interface is wired in lazily, by *replacing the BSAM CHECK routine
address in the extract DCB*:

```asm
OPENSORT LARL  R0,SORTINIT          OVERRIDE BSAM READ ROUTINE ADDRESS
         O     R0,MODE31
         ST    R0,EXTRCHKA
         STCM  R0,B'0111',DCBCHCKA  DCB CHECK  -> SORTINIT
         MVC   DCBBLKSI,H8K         8K "blocks"
OPENNOP  LARL  R0,GETBR14           DCB READ   -> BR R14 (no-op)
         ...
         STCM  R0,B'0111',DCBGETA
         LHI   R15,128              128 buffers
```

So the first time `GVBMR88` "checks" an extract read, control enters
`SORTINIT`, which opens `SYSIN` (`MSG#412`), copies the user's SORT control
cards (`SORT FIELDS=...`) into a `GETMAIN`ed area sized for four records
(`MSG#450` on overflow), and appends the fixed statement

```asm
SORTREC  DC    C' RECORD TYPE=V,LENGTH=8192 '
```

It then re-points the DCB CHECK address to `E35RETRN`, installs `SORTE35`,
obtains a fresh save area for the sort, sets `STATFLG2=insort`, and
`LINK EP=SORT`s. (A hard-coded default statement
`SORTCTRL`/`SORTVERB`/`' FIELDS=(15,'`/`SORTKEYL` exists only for the
alternate `EXITINIT` path and is not reached from `OPENSTD`.)

The sort parameter list is the classic 24-bit list stored in the work area:

```asm
SORTRSA  DS    A          MR88 save area while inside the E35
SORTCTLA DS    A          A(control statements)
SORTE15A DS    A          A(E15)  (only used by DCONINIT)
SORTE35A DS    A          A(SORTE35)
SORTTHRD DS    A          user exit address constant (passed to E35 as 8(R1))
         DS    4A
SORTID   DS    CL4        'MR88'
SORTFFFF DS    XL4        X'FFFFFFFF' terminator
         ...
         LA    R1,SORTCTLA-WORKAREA(,R3)
         LINK  EP=SORT
```

The state flags `insort EQU 01` / `postsort EQU 02` in `GVBMR88W` record
whether report processing is running under the sort's E35 or after SORT
returned.

**`SORTE35` — the coroutine that turns sort output into "BSAM" input.** The
E35 exit is entered by DFSORT with R1 -> (record address, previous record,
user constant). It switches R13 to the saved MR88 save area, copies the
record into the extract buffer, and *emulates a completed BSAM READ* by
posting the DECB and setting the block length so that the normal `READLOOP`
path can consume it unchanged:

```asm
SORTE35  STM   R14,R12,savgrs14
         L     R0,0(,R1)              record address
         LTR   R0,R0
         JP    SORTOUT
         LHI   R15,8                  no more records -> RC 8 (end)
         BR    R14
SORTOUT  LR    R14,R13
         L     R13,8(,R1)             MR88 work area from user constant
         ST    R14,SORTRSA
         ...
         LH    R15,0(,R14)            record length
         L     R1,EXTRDECB
         MVI   4(R1),X'7F'            post ECB "complete"
         L     R1,4+12(,R1)           buffer address from DECB
         LA    R0,4(,R15)
         STH   R0,0(,R1)              block length (BDW)
         XC    2(2,R1),2(R1)
         LA    R0,4(,R1)
         LR    R1,R15
         MVCL  R0,R14                 copy record into block
         LM    R14,R12,savgrs14
         BSM   0,R14                  resume MR88 after its "READ"
```

When MR88 needs the next record it "returns" to DFSORT through `E35RETRN`:

```asm
E35RETRN STM   R14,R12,savgrs14
         cli   statflg3,x'ff'         stop requested?
         be    E35RETRP
         j     e35retrq
E35RETRP mvi   statflg3,x'00'
         LHI   R15,16                 RC=16: terminate DFSORT
         j     e35retrr
E35RETRQ LHI   R15,4                  RC=4 : delete record (already consumed)
E35RETRR L     R13,SORTRSA
         ST    R15,savgrs14+4
         LM    R14,R12,savgrs14
         BSM   0,R14
```

Every record is answered with RC 4 ("delete"), so DFSORT never writes a
`SORTOUT`; MR88 has already processed it. A bad SORT completion is `MSG#431`.
`MSG#499` marks a recursive error raised while inside the E35 (not reprinted).

`DCONINIT` (`LOAD EP=GENXD88`, `DCONGET`) and `EXITINIT` (`LOAD EP=GENXS88`)
are an alternate "direct connect" path that replaces the extract READ/CHECK
addresses (`EXTRCHKA`, `DCBCHCKA`) with a generated routine; they are present
in the source but nothing branches to them: `OPENSTD` only reaches `SORTINIT`
or the direct BSAM open for the accepted `EXTROPT` values `Y`/`N`.

## 10.4 Extract-file record layouts

All three record kinds share the 16-byte prefix (RDW + lengths + view number)
and are distinguished by their content:

```asm
EXTREC   DSECT                    detail record
EXRECLEN DS    HL02               RDW length
         DS    XL02
EXSORTLN DS    HL02               sort-key area length
EXTITLLN DS    HL02               title-key area length
EXDATALN DS    HL02               DT (detail) column data length
EXNCOL   DS    HL02               number of CT (subtotal) columns
EXVIEW#  DS    FL04               view number
EXSORTKY DS   0CL01               sort keys | title keys | DT data | CT columns

HDRREC   DSECT                    header record: HDVIEW# has low bit set
HDSORTKY DS   0CL01               low-values sort key, then HDRDATA
HDRDATA  DSECT
HDRECCNT DS    PL06               extract record count for the view
HDUSERID DS    CL08
HDEVNTNM DS    CL08               event file DDNAME
HDSATIND DS    CL01               request satisfied indicator
HD0C7IND DS    CL01               0C7 abend indicator
HDOVRIND DS    CL01               extract limit exceeded indicator
HDLIMIT  DS    PL06

CTLREC   DSECT                    control record: CTVIEW# = low values
CTRECCNT DS    PL06
CTFILENO DS    HL02
CTPROCDT DS    CL08
CTPROCTM DS    CL06
CTFINPDT DS    CL06               financial period (CCYYMM)

COLEXTR  DSECT                    one CT column inside the data area
COLNO    DS    HL02
COLDATA  DS    PL12
```

The header indicator is the low-order bit of the view number, so a header
sorts immediately before the detail records of its view:

```asm
CHKHDR   L     R0,HDVIEW#
         SRL   R0,1               strip header indicator
         ST    R0,HDVIEW#
         C     R0,SVVIEW#         same view as before?
         JE    EXTPROC
```

## 10.5 The main read loop

```asm
READLOOP llgf  R15,READSUBR       MERGEREC (or sort/direct variant)
         BASSM R10,R15
NEXTEXT  ST    R1,RECADDR
         LR    R7,R1
```

* `MERGEREC`/`MERGNEXT` (addresses `MERGEADR`, `MERGENXT` with `X'80000000'`
  set) were written to merge the extract file with an optional master file;
  in the shipped code `OPENSTD` sets `MSTREOF=C'Y'` unconditionally, so only
  the extract side is read. A non-header record for an unknown view that did
  not come from the master is `MSG#401`, and `GVBMR87` rejects marginal-file
  view definitions with `MSG#449`.
* Before the first record, `GVBMR88` sets the extract DCB EODAD to `EXTREND`
  and reads control records, summing `CTRECCNT`. If header records are expected
  (`NOHDROPT≠Y`) and none is found, `MSG#400`.
* A record whose view number differs from `SVVIEW#` is validated with
  `VALIDHDR`. A valid header triggers `RPTBREAK` for the previous view (grand
  totals), then `HDRINIT` captures `HDRECCNT`, `HDSATIND`, `HD0C7IND`,
  `HDOVRIND` into `VWRECCNT`/`VWSATIND`/`VW0C7IND`/`VWOVRIND`. Consecutive
  headers for the same view are summed (`HDRSUM`, several extract threads each
  write one). A header with no following detail records prints the
  "no extract records" message (`NOEXTREC`).
* `RBRKNEW` locates the `VIEWREC` on `VWCHAIN` (`MSG#402` if absent), resets
  break state, builds report titles (`VWRTADDR`), and falls into `EXTPROC`.
* `EOREOFCD` tracks the reason for the current break: `' '` normal, `'H'`
  reading headers, `'R'` end of request, `'F'` end of file.

## 10.6 `EXTPROC`: processing one detail record

```text
EXTPROC
  |- VWDESTYP = FILEFMT(3) or CSV(7)?
  |     yes, VWSUMTYP=DETAIL(2) -> DETFILE (write detail file record)
  |     yes, summary            -> compare whole sort key (CLCL) with SVSORTKY;
  |                                break -> COMFILE writes the summary record
  |- otherwise (BATCH/ONLINE report)
  |     compare each SORTKEY (EX SRTBREAK CLC) from highest level down;
  |     first differing level -> KEYBREAK with R2 = unchanged levels
  |- normalize CT columns into the accumulator array (EXTCOLA)
  |- copy pre-calculation values to EXTPCALC
  |- DETAIL view  : CALCCOLM -> EXCPCHK -> COLBUILD -> print (DETPRNT)
  |  SUMMARY view : CALCCOLM -> SUM (add into lowest subtotal set)
  '- ST R7,PREVRECA ; J READLOOP
```

### 10.6.1 Normalization into extended DFP accumulators

Each `COLEXTR` in the record carries a column number and a `PL12` packed
value. It is placed into the accumulator array by column number and converted
in place to extended DFP using the quantum that `GVBMR87` left in `fp9`:

```asm
         eextr r12,fp9              biased exponent chosen by MR87
         do from=(r0)
           LH  R14,COLNO
           BCTR R14,0
           SLL R14,2
           A   R14,CLCOFFTB         offset table indexed by column number
           L   R15,0(,R14)
           AR  R15,R1               + CALCBASE
           ZAP 0(AccumDFPl,R15),COLDATA
           lmg r2,r3,0(r15)
           cxstr fp0,r2             signed packed -> DFP extended
           iextr fp0,fp0,r12        insert exponent (decimal places)
           GVBSTX fp0,0(,r15)
           AHI R6,COLDATAL
         enddo
```

`GVBLDX`/`GVBSTX` are the project macros that load/store an FP register pair.
`SUM` adds the current set into the lowest break level set; `FRSTSET`
initializes MIN/MAX/FIRST/LAST accumulators when `VWNOMIN`/`VWNOFRST` are not
set.

### 10.6.2 Sort-key break detection

For report destinations, `EXTINIT` walks the `SORTKEY` array from the highest
level, comparing `SKFLDLEN` bytes at `SKVALOFF` in the current record against
the saved key (`SVSORTKY`):

```asm
EXTSRCH  LA    R14,EXSORTKY
         AH    R14,SKVALOFF
         L     R15,SVSORTKY
         AH    R15,SKVALOFF
         LH    R1,SKFLDLEN
         BCTR  R1,0
         EX    R1,SRTBREAK          CLC 0(0,R14),0(R15)
         JNE   EXTBRK
         AHI   R2,1                 R2 = number of unchanged levels
         AHI   R6,SKENTLEN
         BRCT  R0,EXTSRCH
```

`KEYBREAK` then processes `VWBRKCNT - R2` levels from the lowest up: for each
level it copies pre-calculation results, runs `CALCCOLM` and `EXCPCHK`, prints
the subtotal line (or writes the summary file record via `COMFILE`), rolls the
level's accumulators into the next higher set, resets `SKCOUNT`, and rebuilds
sort-key titles (`SKTITLE`, via the title lookup buffer `SKLBADDR` when the
key has a lookup title). `EXEC`/`PIVOT` outputs (`VWEXEC`, `VWPVIT`) suppress
dashes and page control.

## 10.7 Column calculations and exception logic

VDP 2210 records are compiled by `GVBMR87` into a `CALCTBL` chain per
calculated column, stored at `CDCALCTB`:

```asm
CALCTBL  DSECT
CALCFUNC DS    A                  address of operator routine
CALCTGTA DS    A                  target (result) address
CALCOP2A DS    A                  operand 2 address (column offset for PUSHC)
CALCVALU DS    xl(AccumDFPl)      constant / branch offset
CALCOPER DS    H                  operator code
BRNCH  EQU 1   COMPEQ EQU 2  COMPNE EQU 3  COMPGT EQU 4  COMPGE EQU 5
COMPLT EQU 6   COMPLE EQU 7  PUSHV  EQU 8  PUSHC  EQU 9  ADD    EQU 10
SUB    EQU 11  MULT   EQU 12 DIV    EQU 13 NEG    EQU 14 ABS    EQU 15
```

`CALCCOLM` is a threaded interpreter over this table: it `LM R2,R4` the three
addresses, adds `CALCBASE` to the operand for `PUSHC` (value from another
column), and `BASR R9,R2` into the operator routine; a negative `CALCFUNC`
marks the end and the result in `ACCUMWRK` is copied to `CDCLCOFF`.

```asm
CALCLOOP LM    R2,R4,calcfunc
         LTR   R2,R2
         JM    CALCDONE
         CLI   CALCOPER+1,PUSHC
         JNE   CALCCALL
         A     R4,CALCBASE
CALCCALL BASR  R9,R2
         J     CALCLOOP
```

When a calculation runs is controlled by `CDCALOPT`: `DETCALC EQU 07` (detail
level, default), `BRKCALC EQU 08` (break level), `RECALC EQU 09` (both).
`CALCBRN` implements conditional branches (`CALCVALU` is a signed offset;
-1/-2/-4 select done/skip-column/skip-row via `CALCTEST`).

`EXCPCHK` evaluates the view's exception stack (`VWEXCOND`, from VDP 1210)
with the same interpreter; the row is accepted when the result equals 1:

```asm
EXCPACPT GVBLDX  fp1,=ld'1'
         GVBLDX  fp0,accumwrk
         cxtr  fp0,fp1
         jne   EXCPSKIP
EXCPRETN LM    R2,R9,SAVECALC
         B     4(,R10)            return +4 : accept
EXCPSKIP ... zero all calculated columns with fp8 (DFP zero)
         B     0(,R10)            return +0 : skip
```

Callers therefore always code two branch instructions after
`BRAS R10,EXCPCHK` (skip, then keep).

## 10.8 Column formatting: `COLBUILD` and `GVBDL96`

`COLBUILD` walks the `COLDEFN` array, builds each column into the output line
at `CDCOLOFF`, and formats numeric values by quantizing the DFP accumulator to
the output decimals and converting it to packed:

```asm
COLBPTOP LH    R14,SAOUTDEC        output decimals
         SH    R14,SAVALDEC        - source decimals
         SH    R14,SAVALRND        - rounding
         GVBLDX  fp4,accumwrk
         lcr   r14,r14
         ahi   r14,6176            6176 = biased zero exponent, extended DFP
         iextr fp8,fp8,r14         put exponent into the DFP zero
         qaxtr fp0,fp4,fp8,0       quantize (align decimal point)
         if cxtr,fp0,eq,fp8
           lpdfr  fp0,fp0          -0 -> +0
         endif
         csxtr r14,fp0,0           DFP -> 31-digit packed in R14/R15
         stmg  r14,r15,tempwork
         ...
         EX    R1,COLBZAP          ZAP into output length
         EX    R1,COLBRZAP         ZAP back
         CP    tempwork2(AccumDFPl),TEMPWORK(AccumDFPl)   digits lost?
         JNE   COLBOVER
```

The formatting routine is chosen per column at VDP load time and stored in
`CDFMTFUN`, indexing `FMTFUNTB`:

```asm
FMTFUNTB DC    A(COLBDL96)        +00 - MUST CALL "GVBDL96"
         DC    A(COLBPTOP)        +04 - PACKED TO PACKED
         DC    A(COLBPTOF)        +08 - PACKED TO FIXED (UNSIGNED)
         DC    A(COLBPTON)        +12 - PACKED TO NUMERIC
         DC    A(COLBPTOU)        +16 - PACKED TO NUMERIC (UNSIGNED)
         DC    A(COLBPTOB)        +20 - PACKED TO BINARY
         DC    A(COLBMSK1)..A(COLBMSKL)   +24..+104 - 21 in-line edit masks
```

Only conversions that the in-line routines cannot do (alphanumeric, dates,
arbitrary masks, `FM_FLOAT`, etc.) go to `GVBDL96` through the `DL96AREA`
parameter block (`MAC/DL96AREA.mac`, mirrored by the COBOL copybook
`GVBCDL96`):

```asm
DL96AREA DSECT
SAVALADR DS    ad       source value address
SAVALLEN DS    HL02     source length
SAMSKLEN DS    HL02     mask length
SAMSKADR DS    AL04     mask address (optional)
SAVALFMT DS    HL02     source format (FM_ALNUM..FM_FLOAT)
SAVALCON DS    HL02     source content code
SAVALDEC DS    XL02     source decimals
SAVALRND DS    XL02     source rounding
SAVALSGN DS    CL01     source signed
SAOUTFMT DS    HL02     output format
SAOUTCON DS    HL02     output content code
SAOUTDEC DS    XL02     output decimals
SAOUTRND DS    XL02     output rounding
SAOUTSGN DS    CL01     output signed
SAOUTJUS DS    CL01     output justification (L/C/R)
 ... GVBDL96 private work fields ...
SAFMTERR DS    CL01     formatting error code
DL96LEN  DS    HL02     edited length in/out
DL96RTNC DS    HL02     return code
```

```asm
COLBDL96 LA    R1,DL96LIST
         llgf  R15,GVBDL96A
         BASsm R14,R15
         LTR   R15,R15
         BZR   R10
         CLI   SAFMTERR,X'2'         truncation?
         JNE   COLBERR1              other error -> MSG#409
         CLI   SAVALFMT+1,FM_ALNUM   alphanumeric truncation is allowed
         JNE   COLBOVER
         BR    R10
```

Overflow (`COLBOVER`) fills the column with `VWOVRFIL`, sets
`STATFLG4.STATOVFL`, and at end of run `MSG#451` is logged and the return code
forced to 8. Format codes are the shared `FM_*` equates (`FM_ALNUM X'01'`,
`FM_ALPHA X'02'`, `FM_NUM X'03'`, `FM_PACK X'04'`, `FM_SORTP X'05'`,
`FM_BIN X'06'`, `FM_SORTB X'07'`, `FM_BCD X'08'`, `FM_MASK X'09'`,
`FM_EDIT X'0A'`, `FM_FLOAT X'0B'`).

## 10.9 Output destinations

`VWDESTYP` and `VWSUMTYP` select the output path:

| `VWDESTYP` | Value | Path |
|---|---:|---|
| `BATCH` | 1 | hardcopy report: `DETPRNT`/`KEYBREAK` build 137-byte `VBA` lines into `PRNTLINE` and call `PRNTSUBR` (= `VSAMWRT`) |
| `ONLINE` | 2 | same line building, but `VSAMWRT` writes through the format exit / `GVBTP90` to a VSAM file (source header: "OSI/OSR, CSB/CRR") |
| `FILEFMT` | 3 | `DETFILE` (detail) or `COMFILE` (summary) write fixed/variable records to `VWDCBADR` via `DATAPUTA` |
| `CSV` | 7 | `COMFILE`/`DETFILE` with `VWBLDCSV`: delimiters `VWFLDDEL`, `VWRCDDEL`, `VW_STRING_DELIMITER`; an optional header line of `CDCSVCOL` titles when `VWDHDR=X'01'` and `VWOUTCNT=0`; `COLBUILD` returns via `COLBLEFT` to left-justify |
| `XML` | 9 | file output with XML tagging in `COLBUILD` |

`VWSUMTYP`: `SUMMARY EQU 01`, `DETAIL EQU 02`, `MERGESUM EQU 03`.

`VSAMWRT` (R0 -> print line RDW, R1 -> print DCB) first offers each line to the
view's format exit (`VWPGMADR`) through the `LEINTER` parameter area
(`FMTPVIEW`, `FMTPRECA`, `FMTPARMA`, `FMTPSECT`, run data...). Exit return code
16 stops the run (`MSG#406`). `VWCURSEC` tells the exit which report section
(`'DL'` detail line, titles, subtotals, footers) is being written.

Page control for `BATCH`/`ONLINE`: `VWLINENO`/`VWPAGSIZ` drive `FOOTPRNT` and
`PAGEBRK`; `NEEDTITL` queues sort-key title lines (`TTLPRINT`) for the next
group; `VWZEROSP` suppresses all-zero detail rows using the `NOTNULL` counter.

## 10.10 End of file, control report and return codes

At `READEOF` the last view is closed with `RPTBREAK` (forced subtotal breaks
for every level, then grand totals from `RPTBREAK`/`GRNDINIT`), the end time is
taken with `TIME STCK`, `CTRLRPT` prints the control report to `MR88RPT`
(record counts per view, `VWRECCNT` vs. records read, `vdp_date`/`vdp_time`,
run number), and `CLOSFILE` closes the title-lookup files through `GVBTP90`
(`PAFUNC=CL`), the extract input, each view's output DCB (only if
`DCBOFOPN` is on), and the control report.

Return-code rules in `RTNERROR`/`returne`:

```asm
         CY    R14,abend_msg           ABEND_ON_MESSAGE_NBR match?
         JNE   ERRMSGPR
         DC    XL4'FFFFFFFF'           deliberate S0C1
ERRMSGPR CHI   R14,MSG#414             output record limit: informational
         JNE   ERRMSGPS
         ... print control report, RC=4
ERRMSGPS LA    R0,8                    all other messages: RC=8
```

Messages are written to `MR88LOG` through `GVBUTMSG` (`MSGTYPE=C'L'`). The
message numbers used by the format phase are 400–454 and 499 in
`MAC/GVBUTEQU.mac` (see [13-parameters-ddnames-messages.md](13-parameters-ddnames-messages.md)).

## 10.11 `VIEWREC`, `SORTKEY` and `COLDEFN` quick reference

```asm
VIEWREC  DSECT
VWNEXT   DS    AL04     chain
VWVIEW#  DS    FL04
VWSUMTYP DS    HL02     SUMMARY/DETAIL/MERGESUM
VWDESTYP DS    HL02     BATCH/ONLINE/FILEFMT/CSV/XML
VWFLAG1  DS    XL01     VWPRTDET VWZEROSP VWDWNIND VWDWNONL VWNOMIN VWNOFRST
VWFLAG2  DS    XL01     VWPRINT VWOUTDCB VWCRPIND VWBLDCSV VWEXEC VWPVIT VWNODD VWFXWDTH
VWSRTCNT DS    HL02     sort key count
VWBRKCNT DS    HL02     sort break count
VWCOLCNT DS    HL02     column count
VWDDNAME DS    CL08     output DDNAME
VWFMTPGM DS    CL08     format exit
VWFMTPRM DS    CL32     format exit parameters
VWLINENO DS    HL02  / VWPAGENO DS PL04 / VWPAGSIZ DS HL02 / VWLINSIZ DS HL02
VWEXCOND DS    AL04     exception stack (CALCTBL chain)
VWOUTCNT DS    PL06  / VWLIMIT DS PL06
VWCLCCOL DS    HL02     calculated column count
VWSETLEN DS    HL02     bytes in one set of column totals
VWSKADDR DS    AL04     SORTKEY array
VWLOWSKY DS    AL04     lowest-level SORTKEY
VWCOLADR DS    AL04     COLDEFN array
VWDCBADR DS    AL04     output DCB
VWPGMADR DS    AL04     format exit call address
VWFLDDEL DS    CL01  / VWRCDDEL DS CL01 / VW_STRING_DELIMITER DS C
VWOVRFIL DS    CL48     overflow fill
VWERRFIL DS    CL48     error fill
```

```asm
SORTKEY  DSECT
SKSRTORD DS HL02 / SKCOLSIZ DS HL02 / SKLBLLEN DS HL02 / SKLABEL DS CL48
SKFLDLEN DS HL02  compare length     SKFLDFMT/SKFLDCON/SKFLDDEC/SKFLDRND
SKTTLOFF/SKTTLLEN/SKTTLFMT           title key in the extract record
SKOUTPOS/SKOUTLEN/SKOUTFMT/SKOUTMSK  printed sort title
SKSRTSEQ DS CL01  'A'/'D'
SKHDRBRK/SKFTRBRK/SKDSPOPT           break header/footer/subtotal display options
SKVALOFF DS HL02  offset of this key inside EXSORTKY
SKLBADDR DS AL04  title LKUPBUFR
SKCOUNT  DS PL06  rows in the current break group
SKTITLE  DS CL160 current title text
SKENTLEN EQU *-SORTKEY
```

```asm
COLDEFN  DSECT
CDCOLNO  DS HL02 / CDCLCCOL DS HL02 / CDCLCOFF DS HL02 (accumulator offset)
CDCOLOFF DS HL02  output offset      CDCOLSIZ DS HL02 width-1
CDOUTFMT/CDOUTCON/CDNDEC/CDRNDFAC/CDSIGNED/CDOUTJUS
CDSUBLVL/CDSUBOPT                    subtotal level and option
CDEXAREA/CDDATOFF/CDDATLEN           where the DT value lives in the record
CDMSKLEN DS HL02 / CDDETMSK DS CL48  edit mask
CDCALCNT DS HL02 / CDCALCTB DS AL04  CALCTBL chain
CDFMTFUN DS AL04  index into FMTFUNTB
CDCALOPT DS HL02  DETCALC/BRKCALC/RECALC
CDCSVCOL DS CL146 CSV heading
CDENTLEN EQU *-COLDEFN
```

Related documents: [05-logic-table.md](05-logic-table.md) for how `GVBMR95`
builds the extract records that this phase consumes,
[07-lookups-and-reference-data.md](07-lookups-and-reference-data.md) for the
title lookup tables, and [11-utilities.md](11-utilities.md) for `GVBDL96`,
`GVBTP90` and `GVBUTMSG`.
