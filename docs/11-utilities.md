# 11 - Utility Modules

This document covers the small, single-purpose HLASM service modules that the
main programs call. Each section records the exact parameter list, register
conventions, z/OS services used, return codes, and who calls it in this
repository. Statements that come from source comments rather than executable
code are marked as such.

Source files: `ASM/GVBUR33.asm`, `ASM/GVBUR35.asm`, `ASM/GVBUR39.asm`,
`ASM/GVBURALI.asm`, `ASM/GVBURZTM.asm`, `ASM/GVBUT99.asm`,
`ASM/GVBUTHDR.asm`, `ASM/GVBUTMSG.asm`, `ASM/GVBUTMUE.asm`, `ASM/GVBDAYS.asm`,
`ASM/GVBDL96.asm`, `ASM/GVBSRCHR.asm`, `MAC/GVBAUR35.mac`, `MAC/GVBHDR.mac`,
`MAC/HDRINFO.mac`, `MAC/GVBMSG.mac`, `MAC/GVBMSGDF.mac`, `MAC/GVBMSGGE.mac`,
`MAC/DL96EQU.mac`, `MAC/DL96AREA.mac`, `MAC/GVBSRCH.mac`.

`GVBTP90` and `GVBUR20` are file-access utilities and are documented in
[09-io-handlers.md](09-io-handlers.md) sections 6.1 and 6.2.

## 1. Who calls what

Resolved from `V(...)` address constants and `CALL`/`LOAD` macros in the
repository (not from comments):

```text
                 GVBMR95 ──┬── GVBURALI  (alias detection, at entry)
                           ├── GVBUR35   (SVC 99 allocation of event files)
                           ├── GVBDAYS   (date arithmetic in generated code)
                           ├── GVBSRCHR  (binary-search routines, via SRCHADDT)
                           ├── GVBURZTM  (zIIP/enclave CPU report, at end)
                           ├── GVBDL96   (field formatting)
                           ├── GVBTP90
                           └── GVBUTMSG  (via GVBMSG macro)

                 GVBMR96 ──┬── GVBUTHDR  (control-report page header)
                           ├── GVBUR20   (BSAM disk/tape service)
                           ├── GVBDL96
                           ├── GVBTP90
                           └── GVBUTMSG

         GVBMR87/GVBMR88 ──┬── GVBDL96
                           ├── GVBTP90
                           └── GVBUTMSG

  GVBMRSQ/GVBMRSU/GVBMRAD ─── GVBUR33   ($symbol substitution in SQL / parms)

                 GVBMRBS ─── IDENTIFY EP=GENWRITE  (GVBUR39 resolves GENWRITE
                                                    for user read exits)

                GVBUTMSG ─── GVBUTMUE  (message text table, V(GVBUTMUE))

                 GVBUT99 ─── stand-alone batch step (JCL PARM=)
```

`GVBUR39` and `GVBURZTM` are not referenced by any other module through a
`V()` constant; `GVBUR39` is a stand-alone stub linked into user exits and
`GVBURZTM` is invoked with `CALL gvburztm,...,linkinst=bassm` from `GVBMR95`.

## 2. Common linkage conventions

Every utility below uses standard OS branch-entry linkage unless noted:

```text
R1  -> parameter list (fullword address list)
R13 -> caller's 72-byte save area (or F4SA for GVBDL96)
R14 -> return address
R15 -> entry address on entry, return code on exit
```

The repository uses two register-save-area offset conventions:

```asm
RSABP    EQU   4          back pointer
RSAFP    EQU   8          forward pointer
RSA14    EQU   12         saved R14
RSA15    EQU   16         saved R15
RSA0     EQU   20         saved R0
RSA1     EQU   24         saved R1
```

Return is normally `BSM 0,R14` so the caller's AMODE is restored from the
high-order bit of R14.

## 3. `GVBUR33` - symbolic variable substitution

**Purpose (source code):** scan a character string for `$` and replace each
`$name` that matches an environment-variable table entry with its value,
shifting the remainder of the string left or right as needed.

Addressing: `RMODE ANY`, `AMODE 31`; entered with R15, `USING GVBUR33,R11,R12`
(two 4K bases).

### 3.1 Parameter list

```asm
PARMLIST DSECT
PARMSTRA DS    A          address of the string to scan
PARMSTRL DS    A          address of a halfword string length
PARMENVA DS    A          address of a fullword -> first ENVVTBL entry
```

### 3.2 Environment-variable table entry

```asm
ENVVTBL  DSECT
ENVVNEXT DS    AL04       next entry (0 = end of list)
ENVVNLEN DS    HL02       name length minus 1 (used directly by EX)
ENVVNAME DS    CL16       variable name, including leading '$'
ENVVVLEN DS    HL02       value length minus 1
ENVVVALU DS    CL128      value text
ENVVTLEN EQU   *-ENVVTBL
```

This is the same list that `GVBMR96` builds from `GENENV` (see
[04-gvbmr96-initialization.md](04-gvbmr96-initialization.md)).

### 3.3 Algorithm

```asm
SYMBLOOP CLI   0(R2),C'$'         symbolic variable?
         BRNE  SYMBSCAN
         L     R4,PARMENVA
         L     R4,0(,R4)
SYMBNEXT LTR   R4,R4              end of list?
         BRNP  SYMBSCAN
         LH    R1,ENVVNLEN
         EX    R1,SYMBENVV        CLC ENVVNAME(0),0(R2)
         BRE   SYMBSYMB
         L     R4,ENVVNEXT
         BRC   15,SYMBNEXT
```

When a match is found, `ENVVVLEN - ENVVNLEN` decides the shift direction:

```text
difference == 0 : overwrite in place
difference <  0 : value shorter  -> shift tail LEFT byte-by-byte (SYMBLEFT),
                  blank-fill the vacated tail (SYMBSPAC)
difference >  0 : value longer   -> shift tail RIGHT byte-by-byte from the
                  end (SYMBRGHT) so the string grows within its buffer
```

Then `EX R1,SYMBMVAL` (`MVC 0(0,R2),ENVVVALU`) inserts the value and scanning
resumes after it. The string length halfword is **not** updated; the caller
must size the buffer so that a right shift does not overrun it (the source
computes the shift length as `R3 - R2 - difference`, so the last
`difference` bytes of the original buffer are dropped).

### 3.4 Registers and return

| Reg | Use |
|-----|-----|
| R2 | current scan position |
| R3 | end-of-string address (`PARMSTRA + *PARMSTRL`) |
| R4 | current `ENVVTBL` entry |
| R9 | parameter list |
| R11/R12 | program bases |
| R15 | always 0 on return (`XR R15,R15`) |

Callers: `GVBMRSQ` (SQL text), `GVBMRSU` (HPU control statements), `GVBMRAD`
(Adabas parameter string).

## 4. `GVBUR35` - SVC 99 dynamic allocation / deallocation

**Purpose:** build SVC 99 text units from a fixed-layout parameter block and
issue `DYNALLOC`. Used by `GVBMR95` to allocate event files whose DSN comes
from the VDP (see 09-io-handlers.md section 1.2).

### 4.1 Parameter block `M35SVC99` (`MAC/GVBAUR35.mac`)

```asm
M35SVC99 DSECT
M35FCODE DS    CL1      function code
M35FCDAL EQU   C'1'      - allocate
M35FCDUN EQU   C'2'      - deallocate
         DS    CL1
M35RCODE DS    H        SVC 99 error code (S99ERROR) on failure
M35VLSEQ DS    H        volume sequence number
M35VLCNT DS    H        volume count
M35PRIME DS    H        primary space quantity
M35SECND DS    H        secondary space quantity
M35BLKSZ DS    H        BLKSIZE
M35LRECL DS    H        LRECL
M35KYLEN DS    H        key length (1-255)
M35KEYO  DS    H        key position (1-256)
M35RETPD DS    H        retention period (1-9999)
M35DIR   DS    H        directory blocks
         DS    CL54     reserved
M35DDNAM DS    CL8      DDNAME
M35DSNAM DS    CL46     dataset name
M35MEMBR DS    CL8      member name or relative GDG
M35STATS DS    CL3      status: OLD/SHR/NEW/MOD
M35NDISP DS    CL7      normal disposition
M35CDISP DS    CL7      conditional disposition
M35VLSER DS    CL6      volume serial
         DS    5CL6     reserved for more volsers
M35CLOSE DS    CL1      free at close
M35RECFM DS    CL2      record format
M35TRKS  DS    CL1      space in tracks
M35CYLS  DS    CL1      space in cylinders
M35RLSE  DS    CL1      RLSE
M35UNIT  DS    CL8      unit name
M35DSORG DS    CL4      DSORG
M35RECO  DS    CL4      VSAM organisation
M35EXPDL DS    CL7      expiration date CCYYDDD
         DS    CL282    reserved
M35SYSOU DS    CL1      SYSOUT class
M35SHOLD DS    CL1      SYSOUT HOLD
M35COPYS DS    H        SYSOUT copies
M35OUTLM DS    F        SYSOUT OUTLIM
M35S99LN EQU   (*-M35SVC99)
```

The macro can generate either a `DSECT` (`DSECT=YES`, default) or an in-line
`DS 0D` area.

### 4.2 Control flow

```text
entry
  │ GETMAIN dynamic work area (WORKSLEN), chain save areas
  │ R7 -> M35SVC99, R8 -> S99RB request block, R3 -> text-unit pointer list,
  │ R4 -> current text unit
  ├─ M35FCODE = '2' : S99VERB = S99VRBUN (unallocate), build DDNAME text unit
  ├─ M35FCODE = '1' : S99VERB = S99VRBAL, BAS R9,V01T0100 builds one text
  │                   unit per non-blank/non-zero field (DDNAME, DSN, member,
  │                   status, dispositions, volser, unit, space, DCB attrs,
  │                   DSORG/RECORG, expiry/retention, SYSOUT attributes)
  │                   Each keyword value is validated against a table
  │                   (STATST#, NCDSPT#, DSORGT#, RECFMT#, RECOT#); an
  │                   unknown value -> ABND0100/ABND0102
  ├─ DYNA0100 : LA R1,RBPTR ; DYNALLOC ; MVC M35RCODE,S99ERROR
  ├─ RETI0100 : if SVC 99 returned a DDNAME (DALRTDDN, used when M35DDNAM
  │             was blank) it is copied back into M35DDNAM
  └─ RETURN  : FREEMAIN work area, R15 = return code, BSM 0,R14
```

### 4.3 Return codes and diagnostics

| Condition | Action |
|-----------|--------|
| SVC 99 successful | R15 = 0 |
| SVC 99 failed | `M35RCODE = S99ERROR`; `GVBMSG WTO,MSGNO=DYNALLOC_FAIL` with module name, DDNAME, `S99ERROR` and `S99INFO` in printable hex; R15 = 16 |
| Bad parameter value (unknown STATUS/DISP/DSORG/RECFM/RECORG) | `GVBMSG WTO` with the offending value, then `ABEND 35,DUMP` |

Registers per the source header: R8 = `S99RB`, R7 = `M35SVC99`, R4 = current
text unit, R3 = text-unit pointer list, R13 = dynamic storage. `AMODE 31`,
`RMODE ANY`; the SVC 99 request block and text units live in the GETMAINed
work area.

## 5. `GVBUR39` - `GENWRITE` resolver

`GVBUR39` (`RMODE 24`, `AMODE 31`) is a 20-instruction stub that user read
exits link with so they can call `GENWRITE` (the record-return coroutine that
`GVBMRBS` establishes with `IDENTIFY EP=GENWRITE`, see 09-io-handlers.md
section 2.3) without a hard link-edit dependency:

```asm
GVBUR39  CSECT
         USING GVBUR39,R15
         J     CODE
UR39EYE  GVBEYE GVBUR39
CODE     L     R15,GENWRITE       entry point already resolved?
         LTR   R15,R15
         BNZR  R15                yes - branch straight to it
         STM   R14,R12,RSA14(R13)
         BASR  R11,0
         USING *,R11
         LOAD  EP=GENWRITE        resolve once
         ST    R0,GENWRITE
         LR    R15,R0
         L     R14,RSA14(,R13)
         LM    R0,R12,RSA0(R13)
         BR    R15                tail-call GENWRITE with caller's registers
GENWRITE DC    A(0)
```

Behaviour: first call does `LOAD EP=GENWRITE`, caches the address in its own
CSECT (so the module must be non-reentrant / not shared across address spaces
with different `GENWRITE` identities), restores the caller's registers and
branches to `GENWRITE` as if the exit had called it directly. The source
header states there are no function codes and no return codes of its own.

## 6. `GVBURALI` - alias detection through `CSVINFO`

`GVBMR95` must know whether it was invoked as `GVBMR95E`, `GVBMR95R`, or the
base name (see 03-gvbmr95-extract-engine.md). `GVBURALI` (`AMODE 31`,
`RMODE ANY`, reentrant) answers this by walking the job pack area:

```asm
         mvc   module_name(l'=c'GVBMR95'),=c'GVBMR95'   major name
         mvc   module_name_len,=a(l'=c'GVBMR95')
         CSVINFO FUNC=JPA,TCBADDR=PSATOLD,ENV=MVS,MIPR=(6),
               USERDATA=MYUSERD,RETCODE=INFORC,RSNCODE=INFORS,...
```

`CSVINFO` calls the embedded `MYMIPR` routine once per module in the JPA.
`MYMIPR` maps `MODI_HEADER` (R11), `MODI_1` (R7) and `MODI_5` (R5):

```text
if MODI_1 present
   if module is reentrant (modi_attr2.modi_reent)
      if minor entry (modi_attr2.modi_minor) and MODI_5 present
         and modi_8_byte_major_name == 'GVBMR95'
            alias_name = modi_8_byte_name ; alias_found = on ; R15 = 4 (stop)
   else  (not reentrant: no major/minor info)
      CUSE substring search for 'GVBMR95' inside modi_8_byte_name
      if found and last byte of the name is not blank
            alias_name = modi_8_byte_name ; alias_found = on ; R15 = 4
```

`RETURN (14,12),RC=(15)` — R15 = 0 tells `CSVINFO` to continue, 4 tells it to
stop.

Output contract (source header, verified in `GVBMR95` at the call site):
R0/R1 hold the 8-byte alias name and R15 = 0 when an alias was found;
otherwise `GVBMR95` is returned with R15 = 4. No parameter list is passed:

```asm
         llgf  r15,=v(gvburali)
         bassm r14,r15
         stm   r0,r1,namepgm            save returned name
         if cij,r15,eq,0
           oi  alias,l'alias
         else
           ni  alias,x'ff'-l'alias
         endif
```

`GVBMR95` later refuses to run when the `alias` flag is off (base name used).

## 7. `GVBURZTM` - zIIP / enclave CPU time report

Called at the end of `GVBMR95` when the job ran authorised (`localauth = 'A'`)
so that the zIIP figures collected with `IWMEQTME` can be printed on the
control report:

```asm
         call gvburztm,((2),(3),(4),(5)),plist4=YES,linkinst=bassm,MF=(E,(1))
```

Parameter list (four 4-byte addresses, from the source header):

```text
Pointer1 -> open DCB (RECFM FA, LRECL 133) for the control report,
            or 0 to write the lines with WTO
Pointer2 -> 8-byte STCK value: total zIIP-eligible time
Pointer3 -> 8-byte STCK value: zIIP-eligible time that ran on a CP
Pointer4 -> 8-byte STCK value: total enclave CPU time (CP + zIIP)
```

Each TOD value is converted with `STCKCONV STCKVAL=(rn),CONVVAL=...` into
`HHMMSSthmiju`, then `SRP` shifted and edited to `hhh:mm:ss.th`; percentages
are computed against the total. The fixed report text is:

```text
Description                  HHHH:MM:SS.hh  Percent
===========================  -------------  -------
zIIP-eligible time on zIIP    hhh:mm:ss.th   999.99
zIIP-eligible time on CP      hhh:mm:ss.th   999.99
zIIP-eligible time            hhh:mm:ss.th   999.99
Other time                    hhh:mm:ss.th   999.99
Total enclave CPU time        hhh:mm:ss.th   999.99
```

R0–R14 are unchanged on return, R15 = 0 (source header: no error exits, no
abends, no messages).

## 8. `GVBUT99` - user-abend step

A stand-alone batch program (not called by other modules) whose only job is to
make a JCL step end with a specific user abend, e.g. to force a job to fail
after a condition-code test:

```text
//STEP EXEC PGM=GVBUT99,PARM='nnnn'
```

Behaviour (source):

```asm
PARMLMAX EQU   4                  max PARM length
PARMDEFV EQU   4095               default (and maximum) abend code
...
         L     R3,=A(PARMDEFV)    default
         LH    R15,PARMLEN
         LTR   R15,R15            PARM present?
         CH    R15,=Y(PARMLMAX)   too long?
         EX    R15,DATA_TRT       all numeric?
         CVB   R3,WORKAREA
         C     R3,=A(PARMDEFV)    > 4095?  -> use default
         ...
         WTO   MF=(E,WTO_AREA)    'GVBUT99: ' + job name + message
         ABEND (R3)
```

A missing, non-numeric, over-long or too-large PARM falls back to abend code
4095. The WTO includes the job name obtained from `PSAAOLD -> ASCB`.

## 9. `GVBUTHDR` - standard report header builder

Called by `GVBMR96` (twice: control report and trace report headers) to place
the standard GenevaERS banner into a report buffer.

### 9.1 Parameter list `HEADERPR` (`MAC/GVBHDR.mac`)

```asm
Headerpr     dsect
Pgmtype      ds  a      -> program type text
Pgmtypel     ds  a      -> its length
Pgmname      ds  a      -> program name
Pgmnamel     ds  a      -> its length
Pgm_title    ds  a      -> program title / function
Pgm_title_ln ds  a      -> its length
Rpt_title    ds  a      -> report title
Rpt_title_ln ds  a      -> its length
Rptddn       ds  a      -> DDNAME the report is written to
Buffadd      ds  a      -> output buffer
Bufflgth     ds  a      -> buffer length
Rpt_reccnt   ds  a      -> fullword receiving the number of lines built
headerpr_l   equ *-headerpr
```

Note that this is a list of **addresses**; `GVBMR96` stores each `LA` result
into the list before `bassm R14,R15`.

### 9.2 Output

Each line is written as a 2-byte length followed by text (the same RDW-less
"length prefixed" line format the control-report writer `RPTIT` consumes).
Text constants come from `MAC/HDRINFO.mac`:

```asm
HDR01 DC CL01'~12345678'         first-line carriage marker
HDRT1  DC C'GenevaERS - The Single-Pass Optimization Engine'
HDRT2  DC C'(https://genevaers.org)'
HDRT4  DC C'Licensed under the Apache License, Version 2.0'
HDR05 DC CL30'Performance Engine for z/OS - '
HDR06 DC C'Release PM ',CL8'&SYSPARM.'    <- release from assembler SYSPARM
HDR08 DC CL17'Program ID:      '
HDR09 DC CL17'Program Title:   '
HDR10 DC CL17'Built:           '
HDR12 DC CL17'Executed:        '
HDR14 DC CL17'Report DD Name:  '
HDR15 DC CL17'Report Title:    '
```

The `Built:` line is filled from the assembly date/time symbols captured at
assembly (`bldTIME`/`bldYEAR`), and `Executed:` from `TIME DEC,...,
DATETYPE=YYYYMMDD` at run time. Return codes (source header): 0 = success,
8 = buffer overflow. Registers: R6 = `HEADERPR`, R5 = line count, R4 = output
position.

## 10. Message subsystem: `GVBMSG`, `GVBUTMSG`, `GVBUTMUE`

Almost every diagnostic in the repository goes through one macro and two
modules:

```text
 caller                     GVBMSG macro                GVBUTMSG           GVBUTMUE
 ──────                     ────────────                ────────           ────────
 GVBMSG LOG,MSGNO=..,  ──>  builds GENMSG parameter ──> L 15,=V(GVBUTMSG)  message
        SUBNO=n,SUBx=..     list (MF=L / MF=E)          BASR 14,15         table
                                                        │                  (GVBMSGGE)
                                                        ├─ directory search ◄──┘
                                                        ├─ substitute &&1..&&8
                                                        └─ type L: PUT to log DCB
                                                           type W: WTO
                                                           type F: format only
```

### 10.1 `GVBMSG` parameter list (`MAC/GVBMSG.mac`, `TYPE=DSECT`)

```asm
GENMSG      DSECT
MSGTYPE     DS AL4       request type: C'L' log, C'W' WTO, C'F' format only
MSGGENV     DS A         address of GENENV (for LOG: supplies the log DCB)
MSGDCBA     DS A         explicit log DCB address (alternative to GENENV)
MSGPFX      DS A         -> 3-character prefix (default 'GVB')
MSGNUM      DS AL4       message number
MSGBUFFA    DS A         -> output buffer (RDW + text)
MSGBUFFL    DS AL4       buffer length; on return: true message length
MSG#SUB     DS AL4       number of substitution strings (0-8)
MSGS1PTR    DS A         substitution text 1 address
MSGS1LEN    DS AL4       substitution text 1 length
 ...        (repeated through MSGS8PTR / MSGS8LEN)
GENMSG_L    EQU *-GENMSG
```

The macro enforces: `TYPE` must be `LOG`, `WTO`, `FORMAT` (or `DSECT`);
`MSGPFX` must be exactly 3 characters; `SUBNO` 0–8 with a `SUBn=` for every
counted substitution; `TYPE=LOG` requires `GENENV=` or `LOGDCBA=`.

### 10.2 Message table (`MAC/GVBMSGDF.mac`, `MAC/GVBMSGGE.mac`, `GVBUTMUE`)

`GVBUTMUE` is a three-line program:

```asm
GVBUTMUE GVBMSGGE CASE=Mixed
```

`GVBMSGGE` expands to `GVBMSGDF GVB,TYPE=START`, 340 `GVBMSGDF nnn,'text',
TYPE=x` lines, and `TYPE=END`. `CASE=UPPER` would fold text to upper case at
assembly time; the shipped build is mixed case (hence the "MUE" name).

Each message element is laid out by `GVBMSGDF` as:

```asm
msgt&id  DC    C'&outID&TYPE '          e.g. '0032U '  (number, type, blank)
         DC    C'text with &&1 .. &&8'
ML&ID    EQU   *-MSGt&id                message length
EL&ID    EQU   *-MSG&ID                 element length (incl. MSG#, MSGEL)
```

Message types actually present among the 340 active `GVBMSGDF` entries in
`GVBMSGGE.mac` (a further 9 are commented out): `U` (215), `S` (101), `I` (16),
`W` (8). `GVBMSGDF` also validates `N`, `E` and `C` as legal type letters but
no active message uses them. The type letter is informational text in the message; it does not
change the return code of `GVBUTMSG`.

The table header (`MDSHEAD`) contains `MDSDIRA`/`MDSDIRND` (directory
start/end), `MDSSTRT` (first message), `MDS000` (fallback message 000) and
`MDSPRFLN`. The directory is an array of

```asm
MSGDIR   DSECT
MSGDIR#  DS    A          first message number of a group
MSGDIRO  DS    A          offset of that group from MDSSTRT
```

### 10.3 `GVBUTMSG` algorithm

```text
1. Validate MSGBUFFA != 0                          else R15 = 20
2. Directory search backwards (DIRLOOP) for the largest MSGDIR# <= MSGNUM,
   then linear scan of that group (GRPLOOP) comparing MSG# until match,
   X'FFFF' end marker, or MSG# > MSGNUM              else R15 = 12 and
   message 000 ('&&1 - Message not known') is formatted instead
3. Build the line in MSGBUFFA: 2-byte length (RDW-style), print-control
   blank, prefix (GVB), 'nnnT ', text with &&n replaced by SUBn text
   - missing SUBn text                             -> R15 = 4 (continues)
   - buffer full                                   -> R15 = 8 (truncated)
   - message 000 itself overflowed                 -> R15 = 16
4. MSGTYPE = 'L': if GENENV (or MSGDCBA) supplies an open log DCB -> PUT
                  otherwise the type is changed to 'W'
   MSGTYPE = 'W': WTO TEXT=(R9),MF=(E,MBSWTO) with the 2-byte length prefix
   MSGTYPE = 'F': no output, caller uses the buffer
```

Return codes:

```asm
MBSNOSUB EQU   4     no text for a substitution variable
MBSBUFOF EQU   8     buffer overflow
MBSNOFND EQU   12    message number not found in table
MBSNOFOF EQU   16    message number not found AND message 000 overflowed
MBSNOBUF EQU   20    buffer not specified
```

Message numbers used by the programs are the `EQU`s in `MAC/GVBUTEQU.mac`
(e.g. `DYNALLOC_FAIL`, `VDP_XLT_TIMESTAMP_ERR`, `IO_DRIVER_UNAVAILABLE`); the
full text catalogue is `MAC/GVBMSGGE.mac`. See
[13-parameters-ddnames-messages.md](13-parameters-ddnames-messages.md).

## 11. `GVBDAYS` - days between two `CCYYDDD` dates

`RMODE ANY`, `AMODE 31`. Called from generated code in `GVBMR95` for date
arithmetic.

```asm
PARMLIST DSECT
PLDATE1  DS    A          -> input date 1 (CCYYDDD, 7 zoned digits)
PLDATE2  DS    A          -> input date 2 (CCYYDDD)
PLDAYSBT DS    A          -> fullword result: days between
```

The routine has no dynamic storage: it saves only R14/R15 and R2–R9 into the
caller's save area and uses the doubleword at `RSA8+8` of that save area as
its `DBLWORK`. Algorithm (executable code, `DAYSMAX EQU 365`):

```text
PACK/CVB CCYY of each date -> R2, R3 ; DDD of each date -> R14, R15
leap test:  TM DBLWORK+3,X'03'  (year divisible by 4; no century rule)
            R6 bit 1 = date-1 is leap, R6 bit 2 = date-2 is leap
            R4/R5 = days in year-1 / year-2 (365 or 366)
if year1 == year2 : result = DDD2 - DDD1
else (year1 < year2):
     result = (days left in year-1) + DDD2
            + (intervening years) * 365 + (intervening years) / 4
            + leap-day corrections for remainders of 2 or 3 years
if date-1 > date-2 the same computation runs with the dates swapped
(REVERSE) and the result is negated with LCR R0,R0
ST R0 -> *PLDAYSBT ; R15 = 0
```

The leap-year test is "divisible by 4" only, so years such as 1900 or 2100
would be treated as leap years. The result is signed: negative when date-1 is
later than date-2.

## 12. `GVBSRCHR` - generated binary-search routines

`GVBSRCHR` (`RMODE ANY`, `AMODE 31`) contains no hand-written search code;
the whole CSECT is produced by a local macro:

```asm
         MACRO
         LOOKUPS &ROUTINE=x
&KLEN_N  SETA  1
.SRCHGEN ANOP
         org   *,256                 align each routine on 256 bytes
         &ROUTINE KEYLEN=&KLEN_N,ENTRY=Y
&KLEN_N  SETA  &KLEN_N+1
         AIF   ('&KLEN_N' LE '256').SRCHGEN
         MEND

GVBSRCHR CSECT
         using (thrdarea,thrdend),r13
         LOOKUPS ROUTINE=GVBSRCH             -> S1TBLA ... S256TBLA
```

so there is one routine per key length 1–256, each with an `ENTRY SnTBLA`.
(The prose header of the file says "1 to 156"; the macro loop bound is 256 and
that is what is assembled.) `GVBMR95` assembles the address table `SRCHADDT`
with `GVBSRCH KEYLEN=n,VCON=Y` (`DC V(SnTBLA)`) and PASS1 patches the correct
entry into generated `LK`/`LU` code through the `CSSRCHR` relocation
(06-generated-code.md).

### 12.1 Routine body (`MAC/GVBSRCH.mac`)

```asm
&SLAB.TBLA DS  0D
         ltg   R4,LBLSTFND           same key as last time?
         JNP   &SLAB.INIT
         lgr   R14,r4
         CLC   LKUPKEY+4(&KEYLEN),0(R14)
         be    l'mc_jump(,r10)       yes: return to FOUND address
&SLAB.INIT LG  R4,LBMIDDLE           root of the balanced tree
&SLAB.LOOP lgr R3,R4
         LA    R14,LKUPDATA
         CLC   LKUPKEY+4(&KEYLEN),0(R14)
         JL    &SLAB.TOP             key lower  -> LKLOWENT
         JH    &SLAB.BOT             key higher -> LKHIENT
&SLAB.FND LA   R14,LKUPDATA
         stg   R14,LBLSTFND          remember for next call
         agsi  lbfndcnt,bin1
         b     l'mc_jump(,r10)       return to FOUND
&SLAB.TOP LTG  R4,LKLOWENT
         JP    &SLAB.LOOP
         J     &SLAB.CHK             not found: effective-date check
&SLAB.BOT LTG  R4,LKHIENT
         JP    &SLAB.LOOP
```

For `KEYLEN > 4` the not-found path (`&SLAB.CHK`) performs the effective-date
scan on the last node examined (R3); for key lengths 1–4 there is no room for
an effective date and the routine returns straight to the NOT FOUND address.

Register contract with generated code (see 06-generated-code.md):

| Reg | Meaning |
|-----|---------|
| R2 | literal pool (`litp_hdr`) |
| R3 | last node examined |
| R4 | current `LKUPTBL` node |
| R5 | `LKUPBUFR` (contains `LKUPKEY`, `LBMIDDLE`, `LBLSTFND`, `lbfndcnt`) |
| R10 | return address: `0(R10)` = NOT FOUND, `l'mc_jump(R10)` = FOUND |
| R13 | `THRDAREA` |

`mc_jump`/`mc_disp_l` are computed from a dummy `jlnop` so the macro knows the
length of the branch instruction the generated code places at `0(R10)`.

Return codes stated in the header: 0 success, 8 error.

## 13. `GVBDL96` - field formatting and mask editing

`GVBDL96` is the single formatting engine used by extract (`GVBMR95`,
`GVBMR96` PASS1 stubs `CSdl96call*`) and format (`GVBMR87`/`GVBMR88`) phases.

### 13.1 Entry points and addressing

```asm
GVBDL96  RMODE ANY
GVBDL96  AMODE 31
GVBDL96  CSECT
         using savf4sa,r13
         stmg  R14,R12,SAVF4SAG64RS14   64-bit register save (F4SA)
         sam64
         b     dl96start
         ENTRY GVBDL96X
GVBDL96X AMODE 64                      entry when caller is already AMODE 64
         stmg  R14,R12,SAVF4SAG64RS14
dl96start larl R11,gvbdl96
         lgr   R12,R1                  parameter list
         llgt  R10,PRMADDR             parameter area (DL96AREA)
```

Callers must supply a **144-byte F4SA** save area, not a 72-byte one. Return
is `bsm 0,r14`, so a caller that entered via `GVBDL96` in AMODE 31 is
returned to AMODE 31.

### 13.2 Parameter list

```asm
PARMLIST DSECT
PRMADDR  DS    A          -> PARMAREA (DL96AREA)
TGTADDR  DS    A          -> target (output) area
LENADDR  DS    A          -> halfword: edited length returned
RTNADDR  DS    A          -> halfword: return code returned
```

`DL96AREA` (`MAC/DL96AREA.mac`) carries the source value address/length
(`SAVALADR`, `SAVALLEN`), source format (`SAVALFMT`), content/date code,
decimals, rounding, mask, sign and target attributes, and `SAFMTERR` where the
error code is set.

### 13.3 Source-format dispatch

```asm
         lgh   R15,SAVALFMT
         if cgij,R15,le,12
           sllg R15,r15,2
SRCFMTBL   B   SRCFMTBL(R15)
           BRU ALPHASRC           01 alphanumeric
           BRU ALPHASRC           02 alphabetic
           BRU NUMERIC            03 zoned numeric
           BRU PACKED             04 packed
           BRU SORTDEC            05 sortable packed
           BRU BINARY             06 binary
           BRU SORTBIN            07 sortable binary (sign bit inverted)
           BRU BCD                08 BCD
           BRU ALPHASRC           09 masked numeric
           BRU EDITNUM            0A edited numeric
           BRU DFPINPUT           0B floating point (DFP)
           BRU BADRC              0C 'Geneva number'
         endif
FMTERROR MVI   SAFMTERR,EMFORMAT
```

Sources longer than `MAXSRC` are clamped; a null/zero length returns
immediately with RC 0.

### 13.4 Return codes (`MAC/DL96EQU.mac`)

```asm
EMBAD    EQU   01  invalid data in source field
EMTRUNC  EQU   02  data truncated to fit within field
EMFUNC   EQU   03  invalid function code
EMOUTNUM EQU   04  invalid output area length for numerics
EMOUTFUL EQU   05  output area full (overflow)
EMOCCURS EQU   06  invalid occurrence count
EMFORMAT EQU   07  invalid format code
```

On exit `SAFMTERR` is copied to `*RTNADDR` and R15, and the edited length
(`R3 - TGTADDR`) to `*LENADDR`. `GVBMR88` treats RC 2 on alphanumeric sources
as acceptable truncation and other non-zero codes as overflow
(10-format-phase-gvbmr87-gvbmr88.md).

### 13.5 Date content codes

`DL96EQU.mac` defines the content codes that `GVBDL96` (and the generated
`CF*DATE` compare functions in `GVBMR95`) use to interpret date fields.
Selected values:

```text
 0 NODATE   unspecified            25 CYMDT   CCYYMMDDHHMMSS
 1 YMD      YYMMDD                 30 CYM     CCYYMM
 3 CYMD     CCYYMMDD               31 CCYY
 5 DMY      DDMMYY                 34 POSIX   CCYY-MM-DD HH:MM:SS.TT
 7 DMCY     DDMMCCYY               36 MDCY    MMDDCCYY
 9 YYDDD                           40 C_M_D   CCYY-MM-DD
11 CYDDD    CCYYDDD Julian         50 USDAT   DD MMMMMMMMM CCYY
13 MMDD                            51 EUDAT   MMMMMMMMM DD, CCYY
19 HMST     HHMMSSTT               52 DMOCY   DD-MMM-CCYY
21 HMS      HHMMSS                 54 NORMDATE CCYYMMDDHHMMSSTT
```

Register usage (header, consistent with code): R12 parameter list, R10
parameter area, R11 base, R1 current source pointer, R2 source length, R3
current output position, R6/R7 mask position/length, R8 last digit position,
R9 internal subroutine return.

## 14. Quick reference

| Module | AMODE/RMODE | Entry linkage | Key service | RC values |
|--------|-------------|---------------|-------------|-----------|
| `GVBUR33` | 31 / ANY | R1 -> 3 addresses | none | 0 |
| `GVBUR35` | 31 / ANY | R1 -> `M35SVC99` | `DYNALLOC` (SVC 99) | 0, 16; `ABEND 35` |
| `GVBUR39` | 31 / 24 | tail-call | `LOAD EP=GENWRITE` | passes `GENWRITE`'s |
| `GVBURALI` | 31 / ANY | no parameters; name in R0/R1 | `CSVINFO FUNC=JPA` | 0 alias found, 4 none |
| `GVBURZTM` | 31 / ANY | `CALL ...,plist4=YES` | `STCKCONV`, `CONVTOD` | 0 |
| `GVBUT99` | 31 / ANY | JCL `PARM=` | `WTO`, `ABEND` | abend nnnn (default 4095) |
| `GVBUTHDR` | 31 / ANY | R1 -> `HEADERPR` | `TIME DEC` | 0, 8 |
| `GVBUTMSG` | 31 / ANY | R1 -> `GENMSG` | `PUT`, `WTO` | 0, 4, 8, 12, 16, 20 |
| `GVBUTMUE` | n/a (data) | `V(GVBUTMUE)` | none | n/a |
| `GVBDAYS` | 31 / ANY | R1 -> 3 addresses | none | 0 |
| `GVBSRCHR` | 31 / ANY | generated-code branch, R10 return | none | branch to FOUND / NOT FOUND |
| `GVBDL96` | 31 & 64 / ANY | R1 -> 4 addresses, F4SA | none | 0–7 (`EM*`) |
