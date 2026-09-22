# 09 - I/O Handlers: Event Readers, Extract Writers, Pipes and Tokens

`GVBMR95` never issues an access-method macro for an *event* file itself.
Every event source is fronted by a small "I/O driver" module that is called
once at thread start to open the source and then hands back the address of
its own read routine. The extract side is the reverse: `GVBMR95` owns the
BSAM `WRITE`/`CHECK` loop, and only the destination (disk/tape, in-storage
pipe, in-thread token, dummy) changes what "write" means.

This document covers:

* the driver contract (`EVNTREAD`, `EODADDR`, `GENPARM`),
* each event-file driver (`GVBMRBS`, `GVBMRVK`, `GVBMRSQ`, `GVBMRSU`/
  `GVBMRHPU`, `GVBMRAD`),
* read exits (LE/COBOL and native) and `GENWRITE`,
* the extract-file write path (`WRTEXT`),
* pipes and tokens,
* the general purpose utilities `GVBTP90` and `GVBUR20`.

Sources: `ASM/GVBMR95.asm`, `ASM/GVBMRBS.asm`, `ASM/GVBMRVK.asm`,
`ASM/GVBMRSQ.asm`, `ASM/GVBMRSU.asm`, `ASM/GVBMRHPU.asm`, `ASM/GVBMRAD.asm`,
`ASM/GVBTP90.asm`, `ASM/GVBUR20.asm`, `ASM/GVBUR39.asm`, `MAC/GVBX95PA.mac`,
`MAC/GVBMR95C.mac`, `MAC/GVBMR95W.mac`, `MAC/GVBDECB.mac`.

## 1. Where the drivers sit

```text
                 GVBMR95 thread (one per ES set)
   +-----------------------------------------------------------+
   |  open:   LTACCMTH -> select driver -> BASSM R14,R15        |
   |          driver OPENs, primes, stores EVNTREAD/EODADDR    |
   |                                                           |
   |  loop:   EVNTLOOP: next record inside current block       |
   |            block exhausted -> BASSM R14,EVNTREAD          |
   |            driver refills: RECADDR, GPRECLEN, EODADDR     |
   |          EVNTCODE -> generated code (LTESCODE)            |
   |            ... WR functions -> WRTEXT -> EXTFILE buffers   |
   |                                                           |
   |  eof:    driver returns RECADDR=0 / branches to EVNTEOF   |
   +-----------------------------------------------------------+
        ^                 ^                  ^
        |                 |                  |
   GVBMRBS            GVBMRVK             GVBMRSQ / GVBMRSU / GVBMRAD
   BSAM disk/tape,    VSAM KSDS           DB2 CAF, DB2 HPU, Adabas
   pipes, read exits  keyed sequential
```

### 1.1 Access-method dispatch

`LTACCMTH` (from VDP 0200 `vdp0200_access_method_id`) is switched on at
thread start (`ASM/GVBMR95.asm`, "CALL ASSOCIATED I/O DRIVER TO OPEN EVENT
FILE"):

```asm
         LH    R0,LTACCMTH        LOAD  ACCESS METHOD ID
         select cij,r0,eq
           when   SEQFILE         Sequential file ?
             llgf R15,GVBMRBS
           when   KSDSFILE        VSAM  KSDS   FILE  ???
             llgf R15,GVBMRVK
           when   DB2SQL          DB2   EVENT  FILE  ???
             llgf R15,MRSQADDR
             ... zero -> LHI r14,DB2_SQL_UNAVAILABLE
           when   DB2HPU
             llgf R15,MRSUADDR    ... DB2_HPU_UNAVAILABLE
           when   CALLADA
             llgf R15,MRADADDR    ... ADABAS_UNAVAILABLE
           when   DB2VSAM
             llgf R15,MRDVADDR    ... DB2_VSAM_UNAVAILABLE
           othrwise
             LHI r14,IO_DRIVER_UNAVAILABLE
             MVC  ERRDATA(8),GPDDNAME
             LHI R15,8
             JZ ERRMSG#
         endsel
         LA    R0,LTREPARM        POINT  TO READ EXIT PARAMETERS
         sty   R0,GPSTARTA
         bassm R14,R15            CALL  I/O   SUBROUTINE (INITIALIZE)
         ltgr  R14,R15            SUCCESSFUL  ???
         JNZ   THRDMSG            NO, end thread
```

| `LTACCMTH` | EQU | Driver | Linkage |
|-----------|-----|--------|---------|
| 1 | `SEQFILE` | `GVBMRBS` | strong `V(GVBMRBS)` |
| 3 | `KSDSFILE` | `GVBMRVK` | strong `V(GVBMRVK)` |
| 6 | `DB2SQL` | `GVBMRSQ` | `WXTRN GVBMRSQ` (optional) |
| 7 | `DB2VSAM` | `GVBMRDV` | `WXTRN GVBMRDV` (optional, not in this repo) |
| 16 | `DB2HPU` | `GVBMRSU` | `WXTRN GVBMRSU` (optional) |
| 17 | `CALLADA` | `GVBMRAD` | `WXTRN GVBMRAD` (optional) |
| 2, 5, 8, 10, 11, 15 | `VSAMFILE`, `IMSFASTP`, `EXCPFILE`, `ORACLE`, `SYBASE`, `MSGQUEUE` | none | `IO_DRIVER_UNAVAILABLE` |

Optional drivers are weak externals, so a link-edit that omits `GVBMRSQ`
still produces a usable `GVBMR95`; the address constant is simply zero and
the thread ends with the corresponding `*_UNAVAILABLE` message.

Before the driver is called, `GVBMR95` sets the DCB's DDNAME from
`GPDDNAME` and points the DCBE at its own `EVNTEOF` (end-of-data) and
`SYNADEX0` (I/O error) exits. If `LTFILTYP` is not `PIPEDEV`, the access
method is none of `DB2SQL`/`DB2HPU`/`CALLADA`/`MSGQUEUE`, and `LTDSNAME` is
not blank, the internal `DYNALLOC` routine builds an SVC 99 request from
the VDP 0200 allocation attributes and calls `GVBUR35` (see
[11-utilities.md](11-utilities.md)) so that event files need no DD
statement.

### 1.2 The driver contract

Drivers are called with R13 = the thread's `THRDAREA` (which doubles as the
save area) and R1 = `PARM_AREA`, the `GENPARM` list defined in
`MAC/GVBX95PA.mac`. A driver's *initialization* entry must:

1. `OPEN` (or connect to) the source.
2. Store the address of its *read* routine, with the AMODE-31 bit on, in
   `EVNTREAD`.
3. Optionally normalize the DCB `RECFM` so that `GVBMR95`'s traversal can
   tell fixed (`X'80'`) from variable (RDW) records.
4. Prime the first block (all drivers do) and set:
   * `RECADDR` - 64-bit address of the first record,
   * `GPRECLEN` - current record length,
   * `EODADDR` - 64-bit address one past the last byte of the block.
5. Return R15 = 0, or a message number and non-zero R15 on error.

The *read* routine is entered with `BASSM R14,R15` directly from
`GVBMR95`'s `EVNTBUFR` and must not use R12 (the `GVBMR95` base register);
`GVBMRVK` and `GVBMRSQ` therefore establish their own base with
`larl r11,<csect>`. End-of-file is signalled either by branching to the DCBE
`EODAD` (which `GVBMR95` has pointed at `EVNTEOF`) or by returning with
`RECADDR`/`EODADDR`/`GPRECLEN` all zero.

Two flags in `GENFILE` let a driver influence zIIP scheduling:

```asm
gp_call_srb ds C   If set to Y by i/o routine at open, MR95 schedules an SRB
GP_redrive  ds C   If set to Y, MR95 switches to TCB mode and re-drives the read
```

`GVBMR95` blanks both flags before calling the driver. At `EVNTBUFR` it
switches a zIIP thread from SRB to TCB mode *before* every read unless the
driver has set `gp_call_srb=Y` (or the `RE` row carries `LTNOMODE`), calls
`EVNTREAD`, and if the driver came back with `GP_redrive=Y` it switches to
TCB mode and re-drives the read. None of the drivers in this repository set
either flag, so with `USE_ZIIP=Y` every physical read runs on the TCB and the
thread returns to SRB mode afterwards (see
[08-threading-ziip-recovery.md](08-threading-ziip-recovery.md)).

### 1.3 `GENPARM` / `GENENV` / `GENFILE` (`MAC/GVBX95PA.mac`)

The same three areas are passed to drivers, read exits, lookup exits and
write exits. `GENPARM` is a standard z/OS parameter list of addresses:

```asm
GPENVA   DS    A   ENVIRONMENT  INFO ADDR   -> GENENV
GPFILEA  DS    A   FILE         INFO ADDR   -> GENFILE
GPSTARTA DS    A   STARTUP      DATA ADDR   -> LTREPARM / LTWRPARM / LKPARM
GPEVENTA DS    A   EVENT RECORD PTR  ADDR   -> RECADDR (a pointer to a pointer)
GPEXTRA  DS    A   EXTRACT    RECORD ADDR
GPKEYA   DS    A   LOOK-UP    KEY    ADDR
GPWORKA  DS    A   WORK AREA POINTER ADDR   -> exit's anchor word
GPRTNCA  DS    A   RETURN CODE       ADDR   -> RETNCODE
GPBLOCKA DS    A   OUTPUT BLOCK PTR  ADDR   -> RETNPTR (read exits)
GPBLKSIZ DS   0A   OUTPUT BLOCK SIZE ADDR   -> RETNBSIZ
```

`GENENV` carries `GPTHRDNO`, `GPPHASE`, `GPENVVA` (the `MR95ENVV` table),
the join stack (`GPJSTPCT`, `GPJSTKA`), process date/time and the error
buffer triple (`GP_ERROR_REASON`, `GP_ERROR_BUFFER_PTR`,
`GP_ERROR_BUFFER_LEN`). `GENFILE` starts with `GPDDNAME` ("must be first")
followed by `GPRECCNT` (8-byte count), `GPRECFMT` (`F`/`V`/`D`), `GPRECDEL`,
`GPRECLEN`, `GPRECMAX`, `GPBLKMAX`, `GPLFID`, `gp_call_srb` and
`GP_redrive`.

## 2. `GVBMRBS` - BSAM sequential files, pipes and read exits

`GVBMRBS` (`RMODE 24`, `AMODE 31`) is the default driver and the most
complex one, because it also implements read exits and the reader side of
in-storage pipes.

### 2.1 Buffer ring and read-ahead

`GVBMRBS` builds a circular list of DECB prefixes, one per buffer
(`DCBBUFNO`, or `IO_BUFFER_LEVEL`), each followed by a BSAM `READ ...,SF`
list generated by `GVBDECB TYPE=READ`:

```asm
&PRE.READ  READ  &DECB.,SF,,0,0,MF=L
           DC    AL4(0)      READ EXIT WORK AREA ADDRESS POINTER
           DC    CL8' '      READ EXIT FOR  DDNAME
           DC    A(0)        PIPE "WRITE"   DECB ADDRESS
```

The prefix in front of each DECB holds `DECBNEXT`, `DECBPREV`, `DECBBUFR`,
`DECBIOB` and `DECBR13S` (used by `GENWRITE`, below). After `OPEN` every
buffer's `READ` is started; the read routine `NEWBUFFR` then works as a
classic double-buffer:

```asm
NEWBUFFR LR    R9,R14             SAVE RETURN  ADDRESS
         L     R3,EVNTDECB        LOAD CURRENT DECB   PREFIX ADDRESS
         L     R1,DECBPREV        PRECEDING   BUFFER  AVAILABLE  ???
         LTR   R1,R1
         BNP   EVNTNXTB           NO  - DON'T START   ANOTHER I/O
         XC    DECBPREV,DECBPREV  CLEAR AVAILABLE  INDICATION
         MVC   10(2,R1),DCBBLKSI  RESET BLOCK SIZE IN DECB
         LA    R1,4(,R1)          LOAD  DECB  ADDRESS
         XC    0(4,R1),0(R1)      CLEAR THE   ECB
         llgf  R15,EVNTGETA       CALL  "GET" ROUTINE  (BSAM READ)
         BASSM R14,R15
EVNTNXTB LR    R14,R3             SAVE     CURRENT DECB  PREFIX ADDR
         L     R3,0(,R3)          ADVANCE  TO NEXT DECB  PREFIX
         ST    R14,DECBPREV       INDICATE PRECEDING  AVAILABLE
         ST    R3,EVNTDECB
NEWEVNT  LA    R1,4(,R3)          POINT  TO   DECB
         Llgf  R15,EVNTCHKA       I/O    COMPLETED  SUCCESSFULLY ???
         BASSM R14,R15            CALL CHECK ROUTINE (31-BIT)
         llgt  R6,16(,R3)         LOAD   BUFFER  ADDRESS  FROM   DECB
         TM    DCBRECFM,X'80'     FIXED/UNDEFINED  LENGTH  ???
         BNO   NEWVAR
         STG   R6,RECADDR         Save First Record address
         llh   R0,DCBLRECL
         ST    R0,GPRECLEN
         lgh   R15,DCBBLKSI       COMPUTE BLOCK  LENGTH
         L     R14,20(,R3)        LOAD IOB ADDRESS
         lgh   R0,14(,R14)        get RESIDUAL COUNT (CSW)
         sgr   R15,r0             SUBTRACT RESIDUAL COUNT (CSW)
         agr   R15,r6
         stg   R15,EODADDR
         BSM   0,R9               RETURN
```

The block length of a fixed-format block is `DCBBLKSI` minus the residual
count in the IOB (CSW bytes 14-15), which is how short last blocks are
handled without RDWs. `EVNTGETA`/`EVNTCHKA` normally point at BSAM `READ`
and `CHECK` but are replaced for pipes (`PIPEGET`/`PIPECHK`) and read exits.

The previous buffer is deliberately kept one step behind the current one
(`DECBPREV`) so that generated code can still refer to the *previous* event
record (the `LTPREV*` / "prior record" functions use R3) while the next
block is being read.

`PAGE_FIX_IO_BUFFERS=Y` (`execpagf`) is honoured by setting `DCBEBFXU` in the
DCBE before `OPEN`; the buffers are then obtained by the open exit already
page-fixed.

### 2.2 Read exits (`EVNTSUBR`)

If the `RE` row names an exit (`LTREEXIT` -> `EVNTSUBR`), `GVBMRBS`
`LOAD`s it and rewires the ring:

```asm
           LA  R9,EVNTSUBR
           LOAD EPLOC=(R9),ERRET=P1SUBERR
           ST  R0,EVNTCHKA          exit entry point becomes the "check"
           LArl R14,P1BR14          "get" becomes a no-op (BSM 0,R14)
           ST  R14,EVNTGETA
           LR  R1,R0
           if CLC,lecobol,eq,05(R1)   COBOL SUBROUTINE ???
             LArl R14,LEREADX       call through LE (CEEPIPI)
           else
             LArl R14,P1CALLRX      call natively
           endif
         ...
         ST    R14,EVNTREAD
```

The exit is recognised as COBOL by the LE signature at offset 5 of its entry
point (`lecobol`). Both call paths:

* swap the two buffers (`DECBBUFR` of the current and next DECB) so the
  previous block stays addressable,
* store the buffer address in `RETNPTR` and its maximum size
  (`GPBLKMAX`) in `RETNBSIZ`, pass `GPSTARTA` = `LTREPARM`, `GPWORKA` =
  `LTREWORK`, and hide the real return address and R13 behind the parameter
  list (`GENPARM2`, `GENPARM3` with the high-order bit set),
* call the exit and interpret `RETNCODE`:

```text
   0   normal   - RETNPTR/RETNBSIZ describe a full block
   8   EOF      - if the exit left data in the buffer via GENWRITE the
                  partial block is passed to GVBMR95 first (RXLOCATN),
                  otherwise EOF is raised immediately
   other        - P1CERROR -> READ_EXIT_ERROR message
```

Default attributes for a read exit are `RECFM=VB`, `LRECL=MAXBLKSI-4`,
`BLKSIZE=MAXBLKSI`; the exit can override them through `GPRECFMT`,
`GPRECMAX`, `GPBLKMAX`. LE exits are called through the `LEINTER` area
(`MAC/GVBMR95C.mac`) and `CEEPIPI`, which `GVBMR95` loads once per address
space and initializes once per thread (`PIPIINIT`, `PIPISEQ`) so that COBOL
exits have a Language Environment enclave on every subtask.

### 2.3 `GENWRITE` - record-at-a-time exits as coroutines

A read exit that produces one record at a time calls `GENWRITE` (found via
`GVBUR39`, which `LOAD`s the `GENWRITE` alias identified by `GVBMRBS` with
`IDENTIFY EP=GENWRITE`). `GENWRITE` appends the record to the current
buffer, building an RDW for variable-length files:

```asm
GENWRITE stm   R14,R12,SAVgrs14
         ...
         lg    R14,RECEND         LOAD  TARGET ADDRESS WITHIN BUFFER
         ...                      compute trial next address (+4 for RDW)
         L     R0,DECBBUFR
         AH    R0,DCBBLKSI
         CR    R1,R0              BUFFER FULL ???
         BH    GENWFULL           YES - PASS  BUFFER   TO  "GVBMR95"
         stg   R1,RECEND
         ...                      build RDW, MVCL record into buffer
         lm    R14,R12,SAVgrs14
         Bsm   0,r14
```

When the buffer is full `GENWRITE` does *not* return to the exit. It saves
the exit's R13 in the DECB prefix (`DECBR13S`), marks the DECB complete and
returns - through the registers saved when `P1CALLRX` called the exit - to
`GVBMR95`, which processes the block. On the next read request `EVNTCHKA`
has been switched to `GENWRETN`, which restores the exit's R13 and branches
back into `GENWRITE` at the `MVCL` retry point:

```asm
GENWFULL MVI   DECBECB,X'7F'      INDICATE  NORMAL COMPLETION
         ST    R13,DECBR13S       SAVE READ EXIT'S SAVE AREA ADDRESS
GENWFRST OC    GPRECCNT,GPRECCNT  FIRST BUFFER  ???
         BNZ   GENWNEXT
         LArl  R0,GENWRETN        CHANGE CALLED PROGRAM ADDRESS
         ST    R0,EVNTCHKA
         LArl  R0,P1CALLRX
         ST    R0,EVNTREAD
GENWNEXT LR    R13,R10
         lm    R14,R12,SAVgrs14
         BSM   0,R14              return to GVBMR95's read caller

GENWRETN stm   R14,R12,SAVgrs14
         L     R3,EVNTDECB
         L     R13,DECBR13S       RESTORE CALLER'S RSA ADDRESS
         lm    R14,R12,SAVgrs14
         BR    R15                RETRY   MOVE
```

The exit therefore runs as a coroutine: from its point of view `GENWRITE`
simply returned, while `GVBMR95` consumed a whole block in between. The
exit signals end-of-data by returning 8 from its main entry; `GENWEOF` then
marks the DECB `X'70'`.

### 2.4 Pipes - reader side

A pipe is an extract file with `VDP0200b_ALLOC_FILE_TYPE = PIPEDEV (4)`
whose reader is another ES set in the same run. `GVBMR96` links the
writer's `WRITE` DECBs and the reader's `READ` DECBs pairwise (`LINKPIPE`),
storing the partner DECB address in the extra word appended by `GVBDECB`,
and copies the writer's `EXTLRECL`/`RECFM` into the reader's DCB. No data
set is opened; instead `EVNTGETA`/`EVNTCHKA` become:

```asm
PIPEGET  ...                       "buffer consumed" -> wake the writer
PIPEPOST L     R1,MDLREADL-4(,R2)  LOAD  WRITE EXIT   DECB ADDRESS
         Llilf R0,X'7F000000'
         SR    R15,R15
         CS    R15,R0,0(R1)        THREAD WAITING ???
         BRE   PUTEXIT             NO  - MARK  BUFFER AVAILABLE
         POST  (1),(0)

PIPECHK  ...
PIPEWAIT TM    0(R2),X'40'         already posted?
         BO    CHKRETRN
         WAIT  ECB=(2)             WAIT FOR  DATA BLOCK
CHKRETRN ...  residual = DCBBLKSI - DECBSIZE of the writer's DECB
         CH    R0,DCBBLKSI         EMPTY   BLOCK    ???
         BL    CHKEXIT
PIPEEOF  ... branch to DCBE EODAD
```

The ECB in each DECB is the hand-shake: the writer posts it with the block
length when a buffer is full, the reader `WAIT`s on it, and the reader
resets it to `X'7F000000'` (or `POST`s the writer if the writer is blocked
on a full ring) when it has consumed the buffer. An empty block (length 0)
or an ECB completion code `X'70'` means end-of-pipe (`EOFEVNT=Y`).

Because a pipe reader `WAIT`s on an ECB, `GVBMRBS` cannot run in SRB mode
on that path; `GVBMR95` handles this with `GP_redrive` and pause elements
for the writer side (section 5).

## 3. `GVBMRVK` - keyed VSAM (KSDS) as an event file

`GVBMRVK` (`RMODE 24`, `AMODE 31`) reads a KSDS sequentially by key using
an ACB/RPL pair kept in the thread area:

```asm
         MODCB ACB=(R7),DDNAME=(*,GPDDNAME),MF=(G,LKUPKEY)
         OPEN  ((R7)),MODE=31            (model MODLOPN1)
         MODCB RPL=(R8),ACB=(R7),AREA=(R2),AREALEN=4,MF=(G,LKUPKEY)
         L     R2,EVNTDCBA             INDICATE FIXED FORMAT
         NI    DCBRECFM-IHADCB(R2),X'3F'
         OI    DCBRECFM-IHADCB(R2),X'80'
         L     R15,=A(VSAMREAD+X'80000000')
         ST    R15,EVNTREAD
         BASR  R14,R15                 DO THE PRIMING READ
```

The model ACB is `MACRF=(KEY,SEQ,IN)`; the RPL runs in *locate* mode
(`AREA` is a 4-byte pointer, `AREALEN=4`) so no record is copied.
`VSAMREAD` is one record per call:

```asm
VSAMREAD larl  r11,gvbmrvk
         GET   RPL=(R8)                OBTAIN NEXT RECORD
         SHOWCB RPL=(R8),FIELDS=(RECLEN),AREA=(R6),LENGTH=4
         llgt  R6,VSAMPTR              --> CURRENT RECORD
         STG   R6,RECADDR
         lgf   R0,LRECL
         ST    R0,GPRECLEN
         agr   R0,R6
         stg   R0,EODADDR              one-record "block"
         bsm   0,r9
```

Because `RECADDR+GPRECLEN = EODADDR`, `GVBMR95`'s `EVNTLOOP` calls the read
routine again for every record. At end-of-file the ACB is closed under an
`ENQ`/`DEQ` on `GENEVA`/DDNAME (the same VSAM data set may be a reference
file elsewhere in the job) and `RECADDR`, `GPRECLEN` and `EODADDR` are
zeroed. Errors are reported with the ACB `ERROR` field or the RPL `FDBK`
(`SHOWCB`) in messages `MODCB_ACB_FAIL`, `MODCB_RPL_FAIL`, etc.

## 4. Database drivers

### 4.1 `GVBMRSQ` - DB2 through the Call Attach Facility

`GVBMRSQ` (`RMODE 24`, `AMODE 31`) treats an SQL statement as an event
file. The statement text comes from the VDP 0200 record
(`vdp0200b_DBMS_SQL`, up to 10 238 bytes) and the subsystem from
`vdp0200b_DBMS_SUBSYS` after `GVBUR33` symbol substitution; the plan is
`GVBMRSQ` unless `DB2_SQL_PLAN_NAME` (`EXECSPLN`) overrides it.

Initialization:

```asm
         LARL  R0,DB2FETCH        INITIALIZE READ ROUTINE ADDRESS
         ST    R0,EVNTREAD
         ...
DB2CONN  CALL DSNALI CONNECT   (DBSUBSYS)         -> DB2_CONNECT_FAIL
         CALL DSNALI OPEN      (DBSUBSYS,DB2PLAN) -> DB2_OPEN_THREAD_FAIL
CONNPREP EXEC  SQL PREPARE SQLSTMT INTO :SQLDA FROM :SQLBUFFR
CONNALLO for each SQLVAR: sum data + indicator lengths -> ROWLEN
         GETMAIN R,LV=ROWLEN ; MVCLE clear ; STG R1,RECADDR
         for each SQLVAR: SQLDATA/SQLIND -> slots inside that row buffer
         EXEC  SQL OPEN DB2ROW
```

The `SQLDA` returned by `PREPARE` is used to lay out a *fixed* host-variable
row: every column's data area and null indicator are placed consecutively in
one `GETMAIN`ed buffer, so the logic table sees a flat fixed-length record
whose layout matches the Workbench LR for the "DB2 SQL" logical file. The
read routine is a single-row fetch straight into that buffer:

```asm
DB2FETCH LR    R9,R14
         larl  r11,gvbmrsq
         EXEC  SQL WHENEVER NOT FOUND  GO TO EVNTEOF
         EXEC  SQL FETCH DB2ROW USING DESCRIPTOR :SQLDA
         L     R15,SQLCODE
         ...   non-zero -> DB2_TRANSLATED_SQLCODE via DSNALI TRANSLATE
```

`SQLCODE +100` closes the cursor (`EXEC SQL CLOSE DB2ROW`) and raises EOF.
A statement with no columns is rejected with `SQL_NO_COLUMNS`;
`SQL_PREPARE_FAIL`, `SQL_OPEN_FAIL` and `SQL_FETCH_FAIL` are followed by
`DB2_TRANSLATED_SQLCODE` (the text returned by `DSNALI TRANSLATE`, or
`SQL_XLATE_FAIL` / `DB2_MSG_TEXT_MISSING` when no text is available).

### 4.2 `GVBMRSU` + `GVBMRHPU` - DB2 High Performance Unload

`GVBMRSU` (`RMODE 31`, `AMODE 31`) replaces the SQL fetch loop with IBM
DB2 HPU (`INZUTILB`), which unloads a table in parallel and calls a user
exit for every row. The exit is `GVBMRHPU` (CSECT `INZEXIT`, `RMODE 24`),
following the HPU exit interface documented in its header:

```text
  FUNCTION 1 : initialization  (RC 0 = exit active, 4 = deactivate)
  FUNCTION 0 : process a row   (RC 0 = write row, 4 = discard row)
  FUNCTION 2 : termination
  R1 -> EXTXPLST  (EXTXFUNC, EXTXASQL -> SQLDA, EXTXASSI, EXTXAUSR,
                   EXTXATID, EXTXAUWA, EXTXEXUE, EXTXUMSG ...)
```

`GVBMRSU` requires APF authorization (`TESTAUTH`, otherwise
`DB2_HPU_NAPF`), writes the HPU control statements to `SYSIN`
(`DB2_HPU_SYSI` on failure), then:

1. builds an `EXUEXU` table (`MAC/GVBMR95C.mac`) with one entry per HPU
   sub-thread and publishes its address with name/token services
   (`DB2_HPU_TOKN` on failure) so each `INZEXIT` instance can find it;
2. `IDENTIFY EP=GVBMRSB` and `ATTACH EP=GVBMRSB,ECB=WKECBSUB` - the subtask
   simply `LINK EP=INZUTILB` and records its return code (`WKSUBERR`);
3. sets `EVNTREAD` to its own `DB2FETCH`, which is a block hand-over: it
   `WAIT`s on the `EXUECBMA` ECB list (`WAIT 1,ECBLIST=`), finds the exit
   instance whose `EXUSTAT` says "buffer filled", exposes that instance's
   block (`EXUBLKA`, `EXURNUM`, `EXUROWLN`) as `RECADDR`/`EODADDR`, and
   `POST`s `EXUECBEX` when the block has been consumed so the exit instance
   can refill it;
4. treats `WKECBSUB` being posted with an error code as `DB2_HPU_FAIL`; RC 4
   from HPU means "all records passed to MR95".

```asm
EXUEXU   DSECT
EXUOUTBN DS    A       association to DB2 HPU sub task
EXUECBMA DS    XL4     ECB THAT MAIN TASK WAITS ON
EXUECBEX DS    XL4     ECB THAT EXITS WAIT ON
EXURPOS  DS    A       CURRENT POSITION OF RECORD IN BLK
EXURLAST DS    A       POSITION OF LAST BYTE IN DATABLOCK
EXUROWLN DS    F       CALCULATED ROW LENGTH
EXUEOF   DS    X       This instance returned final data block
EXUWAIT  DS    X       This instance filled buffer and waiting
EXUSTAT  DS    X       1: filling  2: filled  3: being processed by MR95
EXURNUM  DS    F       Number records in returned block
EXUBLKA  DS    A       Address data block
```

Like `GVBMRVK`, the driver forces `RECFM=F` in the DCB and `GPRECFMT=C'F'`.
Access method 16 is selected by the Workbench in the VDP 0200 record; the
`MR95PARM` keyword `UTILITY` (`EXEC_DB2HPU`, `Y`/`N`) is parsed and echoed
in the control report by `GVBMR96` but does not itself change the access
method. The message numbers are `DB2_HPU_UNAVAILABLE` (198),
`DB2_HPU_FAIL` (199), `DB2_HPU_NAPF` (201), `DB2_HPU_SYSI` (202),
`DB2_HPU_PARA` (203) and `DB2_HPU_TOKN` (204).

### 4.3 `GVBMRAD` - Adabas

`GVBMRAD` (`RMODE 31`, `AMODE 31`) reads an Adabas file with `L3`
(read logical sequential) commands through `ADAUSER` (`LOAD EPLOC=LINKNAME`,
`ADA_NOLINK` if it cannot be loaded), using the ACBX (extended control block)
and buffer descriptors (`FBDX`, `SBDX`). Its "SQL text" is a keyword string
parsed from the VDP 0200 record:

```text
  SB=<search buffer, 3-8 bytes>   FB=<format buffer, 3-256 bytes>
  DBID=<1-65535>  FNR=<file number>  LREC=<record length>
```

each validated with its own message (`ADABAS_SBL`, `ADABAS_FBL`,
`ADABAS_DBID`, `ADABAS_FNR`, `ADABAS_LREC`). After an `OP` command the read
routine issues `L3` calls with multifetch (`HMISN` records per call, buffer
sized as `HMISN * LREC`) and exposes the returned record block as a
fixed-format block to `GVBMR95`; non-zero Adabas response codes are reported
with `ADA_BADRSP`.

## 5. Extract writing (`WRTEXT`)

All `WR*` logic-table functions end in `WRTEXT` (or `WRTSUM` for
summarized views, or the token variants below). Sequence
(`ASM/GVBMR95.asm` from label `WRTEXT`):

1. Compute the record length: `R8 - R7` (current column pointer minus
   record start), raised to `EXTMINLN`; store in the RDW (`EXRECLEN`), the
   column count in `EXNCOL`, `EXVIEW#` and the sort/title/data lengths
   from the `NV` row.
2. If the `WR` row has a write exit (`LTWRADDR`), call it with
   `GPSTARTA=LTWRPARM`, `GPWORKA=LTWRWORK`, `RETNPTR=R7` and interpret
   `RETNCODE`: 0 write (possibly a substituted record in `RETNPTR`),
   8 skip, 12 disable the view (`LTSTATUS=C'W'`, `DISABREQ`), >= 16 abort
   (`ABORTEX`).
3. Serialize on the `EXTFILE` with `EXTINUSE` (CS + pause element, see
   [08-threading-ziip-recovery.md](08-threading-ziip-recovery.md)).
4. `WRTXNEW`: if the record does not fit in the current buffer, flush it -

```asm
           llgt R1,EXTDECBC        LOAD  CURRENT DECB PREFIX ADDRESS
           llgt R15,16(,R1)        LOAD   BUFFER ADDR FROM   DECB
           sgr R4,R15             COMPUTE BLOCK LENGTH
           if TM,EXTRECFM,X'40',o     VARIABLE   OR UNDEFINED   ???
             STH R4,0(,R15)       BUILD   BLOCK DESCRIPTOR  WORD  (BDW)
           endif
           STH R4,10(,R1)         PLACE  LENGTH IN    DECB
           ltgf R15,EXTDCBA        DCB ALLOCATED ???
           JNP WRTXPIPE           NO  - CHECK IF PIPE
           TM  48(R15),X'10'      EXTRACT FILE  STILL OPEN  ???
           JO  WRTXPUT
           MVI 4(R1),X'7F'        NO  -  PRETEND  I/O COMPLETED (dummy)
           ...
WRTXPUT    aghi R1,4               POINT TO DECB (FOLLOWING PREFIX)
           XC  0(4,R1),0(R1)      CLEAR ECB
           llgf R15,EXTPUT_6431    LOAD  WRITE  ROUTINE ADDRESS
           bassm R14,R15           WRITE PHYSICAL BLOCK (31-BIT MODE)
WRTXSKIP   llgt R1,EXTDECBC        advance to next DECB prefix
           ...
           llgf R15,extchk_6431    LOAD  CHECK  ROUTINE ADDRESS
           bassm R14,R15
WRTXNOWT   ... EXTRECAD = buffer (+4 for BDW if variable)
           ... EXTEOBAD = buffer + EXTBLKSI
```

   `EXTPUT_6431`/`EXTCHK_6431` are AMODE-31 stubs around BSAM `WRITE`/
   `CHECK` (generated code runs in AMODE 64). If zIIP is active the flush
   is preceded by `TCB_switch` (and the thread later returns to SRB mode
   with `SRB_switch`), because BSAM cannot be issued from an SRB. The
   `TM 48(R15),X'10'` test is `DCBOFLGS`/`DCBOFOPN` - a DCB that was never
   opened (dummy output) or has already been closed makes the write a
   no-op. `WRTXPIPE` handles `EXTDCBA = 0`: if the VDP 0200 record has a
   `FILE_READER` the block goes to the pipe routines, otherwise it is
   dropped.
5. Copy the record to `EXTRECAD`, bump `EXTCNT`/`EXTBYTEC` and the
   thread/view counters, release `EXTINUSE`.

The `EXTFILE` control block (`MAC/GVBMR95C.mac`) is shared by every thread
writing to that DDNAME:

```asm
EXTFILE  DSECT
EXTDDNAM DS    CL8             EXTRACT FILE DDNAME
EXTVDPA  DS    FDL08           EXTRACT FILE VDP 200  RECORD   ADDRESS
EXTDCBA  DS    A               EXTRACT FILE DCB      ADDRESS  (0 = pipe/token)
EXTDECBF DS    A               EXTRACT FILE FIRST    DECB     ADDRESS
EXTDECBC DS    A               EXTRACT FILE CURRENT  DECB     ADDRESS
EXTPUTA  DS    A               EXTRACT FILE WRITE    ROUTINE
EXTCHKA  DS    A               EXTRACT FILE CHECK    ROUTINE
EXTPRINT DS    A               EXTRACT FILE PRINT    NEXT     POINTER
EXTINUSE DS    A               EXTRACT FILE IN-USE   POINTER
EXTEOBAD DS    A               CURRENT END-OF-BUFFER ADDRESS
EXTRECAD DS    A               CURRENT      RECORD   ADDRESS
EXTRECLN DS    HL2
EXTCNT   DS    XL8             EXTRACT FILE RECORD COUNT
EXTBYTEC DS    xL8             EXTRACT FILE BYTE COUNT
EXTRECFM DS    XL2
EXTMINLN DS    HL2             EXTRACT FILE MINIMUM  RECORD   LENGTH
EXTLRECL DS    HL2
EXTPIPEP DS    HL2             PARENT THREAD  COUNT  ("PIPED INPUT")
EXTPIPED DS    HL2             THREAD  DONE   COUNT  ("PIPED INPUT")
EXTPIPEC DS    HL2             CHILD  THREAD  COUNT  ("PIPED INPUT")
EXTBUFNO DS    HL2
EXTBLKSI DS    HL2
extflag  ds    x
extfmtph equ   x'80'            This ddname is used in a Format Phase
EXTPUT_6431  DS    A           extract file amode 64 WRITE    ROUTINE
EXTCHK_6431  DS    A           extract file amode 64 check    ROUTINE
EXTFILEL EQU   *-EXTFILE
```

### 5.1 Extract record layout

```text
   +----+----+----+----+----+----+--------+--------------+--...--+--------+
   |RDW (EXRECLEN,0)   |SORTLN|TITLLN|DATALN|NBR CT COLS|VIEW ID|sort key|
   +----+----+----+----+----+----+--------+--------------+--...--+--------+
   |  title key | DT column data area | CT columns (COLDATAL each)        |
   +------------+---------------------+------------------------------------+
```

`GP_EXTRACT_REC` in `MAC/GVBX95PA.mac` documents the same prefix (`GP_SORT_KEY_LENGTH`,
`GP_TITLE_KEY_LENGTH`, `GP_DATA_AREA_LENGTH`, `GP_NBR_CT_COLS`, `GP_VIEW_ID`).
The format phase (`GVBMR88`) relies on this prefix to find the sort key and
the CT (calculated/accumulated) columns.

### 5.2 Pipes - writer side

For a `PIPEDEV` extract file `EXTDCBA` is zero and `EXTPUT_6431`/
`EXTCHK_6431` point at `PIPEPUT`/`PIPECHK` in `GVBMR96`:

```asm
PIPEPUT  LT    R1,MDLWRTL-4(,R2)  LOAD READ EXIT DECB ADDRESS (partner)
         JNP   PUTEXIT
         iilf  R0,x'7F000000'
         XR    R15,R15
         CS    R15,R0,0(R1)       THREAD WAITING ???
         JE    PUTEXIT            NO  - MARK  BUFFER AVAILABLE
         POST  (1),(0)            YES - wake the reader

PIPECHK  TM    0(R2),X'40'        buffer already released by reader?
         BRO   PIPECHKR
         WAIT  ECB=(2)            WAIT FOR BUFFER TO BE EMPTIED
```

A pipe may have several writer threads (`EXTPIPEP` parents) and the reader
must not see EOF until all have finished. `PIPEEOF` (called at each writer
thread's `ES` end) increments `EXTPIPED` with a `CS` loop and, when
`EXTPIPED >= EXTPIPEP`, writes the final partial block followed by an empty
block that the reader interprets as end-of-pipe. `EXTPIPEC` counts the
child (reader) threads for the control report.

### 5.3 Tokens

A token (`TOKENDEV = 5`) is a pipe without buffers: the writing view and the
reading view(s) run *in the same thread*, and each record is delivered
synchronously. `WRTK` (`MDLWRTK`) copies the extract data area into a
`LKUPBUFR` that plays the role of the "token record":

```asm
MDLWRTK  llgt  R5,0(,R2)                 LOAD LOgic table row addr
         ...   locate the RETK's literal pool through lp_base_litp
         lgf   r5,ltwrlubo-logictbl(,r5) get lkupbufr offset
         llgt  r5,0(r5,r2)               get the buffer itself
         sty   r5,4-524288(r15,r3)       Save lkup bufr for RETK
         agsi  0(r14),bin1               Increment count
         LA    R14,EXSRTKEY-EXTREC(,R7)  LOAD EXTRACT RECORD ADDRESS
MDLWRTKA aghi  R14,0                     ADVANCE TO DATA AREA (CSSRTLEN)
         lgh   R15,LBRECLEN-LKUPBUFR(,R5)
         ...
         MVCL  R0,R14                    MOVE DATA AREA TO TOKEN
MDLWRTKB ...  increment LTWRCNTI / LTWRCNTO
MDLWRTKC EQU   *                         CSCALLVW: call the reading views
```

The `CSCALLVW` relocation at `MDLWRTKC` is filled by `GVBMR96` with a
`callview` code sequence that branches to the generated code of each view
reading the token (`RETK` rows, chained through `LTNXRTKN`) and returns.
`WRTX` (`MDLWRTX`) is the same with a write exit in front (`WRTTKN`); an
exit return code of 8 skips the call-view sequence. The `ET`
(end-of-token) function terminates the reading views' code so control
returns to the writer. Because tokens never leave the thread there is no
serialization and no buffer flush; only counters are maintained.

### 5.4 Dummy extract files

When `TREAT_MISSING_VIEW_OUTPUTS_AS_DUMMY=Y` a view whose `EXTRnnn` DD is
missing gets a `DUMYnnn` DCB that is never opened. `WRTXNEW` detects this
(`TM 48(R15),X'10'` - DCB not open) and marks the DECB complete without
I/O, so the buffer is simply reused and records are discarded while the
counters keep counting.

## 6. General-purpose I/O utilities

### 6.1 `GVBTP90` - dynamic VSAM/QSAM interface

`GVBTP90` (`RMODE ANY`, `AMODE 31`) is a reentrant, COBOL-callable file
server for files whose DDNAMEs are not known until run time. `GVBMR96` and
`GVBMR87` use it to read the VDP and (in some paths) reference files; user
exits may call it too. Parameters are three addresses:

```asm
PARMLIST DSECT
PARMADDR DS    A               ADDRESS OF PARAMETER AREA
RECADDR  DS    A               ADDRESS OF RECORD    AREA
KEYADDR  DS    A               ADDRESS OF KEY       AREA

PARMAREA DSECT
PANCHOR  DS    AL04            STORAGE ANCHOR (file control block chain)
PADDNAME DS    CL08            FILE    DDNAME
PAFUNC   DS    CL02            FUNCTION CODE
PAFTYPE  DS    CL01            FILE TYPE (V = VSAM, S = SEQUENTIAL)
PAFMODE  DS    CL02            FILE MODE (I, O, IO, EX)
PARTNC   DS    CL01            RETURN CODE
PAVSAMRC DS    HL02            VSAM   RETURN CODE
PARECLEN DS    HL02            RECORD LENGTH
PARECFMT DS    CL01            RECORD FORMAT (F, V)
PARESDS  DS    CL01            ESDS direct
```

| `PAFUNC` | Meaning | | `PARTNC` | Meaning |
|----------|---------|-|----------|---------|
| `OP` | open | | `0` | successful |
| `CL` | close | | `1` | not found (`RD`) |
| `RD` | read by full key | | `2` | end-of-file (browse) |
| `LO` | locate (generic key, `>=`) | | `B` | bad parameter |
| `SB` | start browse | | `E` | I/O error |
| `BR` | browse next | | `L` | logic error (not open, read past EOF) |
| `WR` | write | | | |
| `UP` | update previous record | | | |
| `DL` | delete previous record | | | |
| `IN` | file information | | | |
| `RI` | release held record (VSAM) | | | |

Each open file gets a file control block chained from `PANCHOR`, holding the
ACB/RPL or DCB, so one parameter area can serve many files.

### 6.2 `GVBUR20` - BSAM disk/tape with block-level access

`GVBUR20` (`RMODE 24`, `AMODE 31`) is the block-oriented sequential I/O
routine used by the report phase and utilities where `QSAM` buffering is not
wanted. Its parameter list is `MAC/GVBUR20P.mac`:

```asm
&PRE.FC   DS    HL02   FUNCTION CODE
&PRE.RC   DS    HL02   RETURN   CODE
&PRE.ERRC DS    HL02   ERROR    CODE
&PRE.RECL DS    HL02   RECORD   LENGTH
&PRE.RECA DS    AL04   RECORD   AREA      ADDRESS
&PRE.RBN  DS    FL04   RELATIVE BLOCK     NUMBER
&PRE.DDN  DS    CL08   FILE     DDNAME
&PRE.OPT1 DS    CL01   I/O MODE (I=IN,O=OUT,D=DIRECT,X=EXCP)
&PRE.OPT2 DS    CL01
&PRE.NBUF DS    HL02   NUMBER   OF I/O    BUFFERS
&PRE.WPTR DS    AL04   WORK     AREA      POINTER
&PRE.MEMS DS    fL04   Size of memory used by GVBUR20
```

Functions: `0` open, `4` close, `8` read sequential, `12` read direct (by
relative block number), `16` write record, `20` write block locate mode,
`24` write block move mode. Return codes: `0`, `4` warning, `8` EOF, `16`
permanent error, with `ERRC` 1-10 distinguishing bad work-area pointer,
undefined function, undefined mode, already open, open failure
(output/input/EXCP), never opened, already closed and bad length.

## 7. Cross-references

* Access-method constants and `PHYSICAL FILE TYPES` (`STDEXTR`, `DISKDEV`,
  `TAPEDEV`, `PIPEDEV`, `TOKENDEV`): `MAC/GVBMR95C.mac`.
* Thread-type selection (disk/tape/other queues) and the SRB/TCB switching
  around I/O: [08-threading-ziip-recovery.md](08-threading-ziip-recovery.md).
* How `GVBMR96` builds `EXTFILE` blocks, DECB rings and pipe links
  (`OPENEXTF`, `LINKPIPE`, `PIPELIST`):
  [04-gvbmr96-initialization.md](04-gvbmr96-initialization.md).
* Read-exit, write-exit and lookup-exit parameter conventions:
  [12-control-blocks.md](12-control-blocks.md).
* Dynamic allocation (`GVBUR35`) and symbol substitution (`GVBUR33`) used
  before `OPEN`: [11-utilities.md](11-utilities.md).
