# 07 - Lookups and Reference Data

This chapter describes how the Performance Engine loads reference ("lookup")
data into memory, how the extract phase (`GVBMR95`/`GVBMR96`) searches it from
generated code, how multi-step joins and effective dates work, and how the
format phase (`GVBMR87`/`GVBMR88`) reuses the same table layout for title
lookups. Every structure and instruction sequence below is taken from
`MAC/GVBMR95C.mac`, `MAC/GVBSRCH.mac`, `ASM/GVBSRCHR.asm`, `ASM/GVBMR96.asm`,
`ASM/GVBMR95.asm` and `ASM/GVBMR87.asm`.

## 1. Data flow

```text
  Reference phase (GVBMR95R + reference logic table)
        |
        |  writes reference data files (one per lookup LR) plus a header file
        v
  +------------------------+      +--------------------------------------+
  | REH  header records    |      | RED data files  (GREFnnn / REFRnnn)  |
  | (TBLHEADR, 1 per file) |      | fixed-length rows, key at TBKEYOFF   |
  +-----------+------------+      +------------------+-------------------+
              |                                      |
   EXTRREH / REFRREH (GVBMR96 LOADLKUP)              |  GVBUR20 reads
   REFRRTH          (GVBMR87 FILLLKUP)               |
              v                                      v
  +---------------------------------------------------------------------+
  | 64-bit reference pool (IARV64 GETSTOR)                              |
  |   LKUPBUFR prefix ---> [LKUPTBL prefix + row] [LKUPTBL prefix + row]|
  |                        ... optional hash index (LBHASHBEG..END)     |
  +---------------------------------------------------------------------+
              ^
              |  generated LUSM / LUEX code, R5 -> LKUPBUFR, key in LKUPKEY
              |
  extract-phase threads (GVBMR95) and format-phase title lookups (GVBMR88)
```

Two DDNAME conventions are used, selected by the executing alias:

| Phase / program | Header file DDNAME | Data file DDNAMEs |
|-----------------|--------------------|-------------------|
| `GVBMR95E` (extract) via `GVBMR96 LOADLKUP` | `EXTRREH` | `GREFnnn` (nnn/nnnn = `TBXFILE#`) |
| `GVBMR95R` (reference) via `GVBMR96 LOADLKUP` | `REFRREH` | `GREFnnn` |
| `GVBMR87` (format-phase init) `FILLLKUP` | `REFRRTH` | `REFRnnn` |

`GVBMR96` switches the header DDNAME with:

```asm
         if clc,namepgm,eq,=cl8'GVBMR95R'    R for reference
           mvc  dcbddnam,=cl8'REFRREH'     default is EXTRREH
         endif
```

and both loaders build the data DDNAME from the header's extract-file number
(three digits up to 999, otherwise four):

```asm
         MVC   UR20DDN,GREFXXX
         LH    R0,TBXFILE#                 CONVERT FILE NO. TO DEC
         CVD   R0,DBLWORK
         OI    DBLWORK+L'DBLWORK-1,X'0F'
         CHI   R0,999                      THREE OR FOUR DIGIT ???
         JH    *+14
         UNPK  UR20DDN+4(3),DBLWORK        BUILD DDNAME SUFFIX (3)
```

## 2. Reference header record (`TBLHEADR`, "REH"/"RTH")

Each header record describes one reference data file. `GVBMR96` copies the
first `TBHRECL` bytes of every record into an in-storage table (`rehtbla`,
eyecatcher `REHTABL`) and then fills the working fields that follow.

```asm
TBLHEADR DSECT                 TABLE   DATA   HEADER RECORD
TBFILEID DS    FL04            FILE    ID
TBRECID  DS    FL04            LOGICAL RECORD ID
TBRECCNT DS    FL04            RECORD  COUNT
TBRECLEN DS    HL02            RECORD  LENGTH
TBKEYOFF DS    HL02            KEY     OFFSET
TBKEYLEN DS    HL02            KEY     LENGTH
TBXFILE# DS   0HL02            "JLT"   EXTRACT  FILE NUMBER
TBDSAMRD DS    CL01            "DSAM"    READ INDICATOR (VERSION 3)
TBEFFIND DS    CL01            EFFECTIVE DATE INDICATOR (VERSION 3)
TBABOVET DS   0FL04            ABOVE   THRESHOLD RECORD  COUNT (V3)
TBREDVER DS    CL01            "RED"   VERSION  (SET BY MR96)
TBEFFDAT DS    XL01            EFFECTIVE DATE OPTION CODE
TBADJPOS DS    XL01            ADJUST  STARTING POSITIONS (N/Y)
TBTXTFLG DS    XL01            TEXT DATA FLAG
TBHRECL  equ   *-TBLHEADR         length of record data needed
TBREFMEM ds    fdL8            Memory needed for all refernce data
TBHPRIME DS    FDL8            PRIME number used for hash table size
TBHMULT  DS    XL04            Hash table multiplying factor
TBINDEX  DS    CL1             Reference data index type
TBBINARY EQU   C'B'              Ref data index Bin tree
TBHASH   EQU   C'H'              Ref data index hash table
TBHASHT  DS    CL1             hash type
TBHDRLEN EQU   *-TBLHEADR         TABLE HEADER LENGTH
```

Validation performed by `LOADLKUP`:

* The expected number of header records is the count of VDP 0650 `GREF`
  entries flagged `vdp0650b_gref_ephase` (`grefcnt`). Reading more or fewer
  records than that ends with `REH_COUNT_ERR` (message 164), with both counts
  formatted into the message.
* A header whose `TBFILEID` begins with `$$` is a Version 3 custom REH and is
  rejected with `REH_FILE_DATA_ERR` (message 137, "Reference-phase work file
  header (EXTRREH) contains invalid data"). Everything else is tagged
  `TBREDVER=C'4'`.
* A header with `TBRECCNT <= 0` still gets a lookup buffer ("placeholder",
  `FILLOCLB`) but no data file is opened.
* After all data rows are read, `GVBUR20` is called once more; anything other
  than return code 8 (end of file) means the header count was wrong
  (`FILLEPTE`) or a read error occurred (`REF_READ_ERR`).

## 3. Memory layout

### 3.1 Lookup buffer prefix (`LKUPBUFR`)

One `LKUPBUFR` exists per (file id, LR id) pair per thread. It anchors the
in-memory table, carries the search state used by generated code, and doubles
as the parameter list for lookup exits.

```asm
LKUPBUFR DSECT                 LOOK-UP BUFFER AREA PREFIX
LBNEXT   DS    AL04            NEXT    BUFFER POINTER (0 = END-OF-LIST)
LBLEN    DS    HL02            THIS    LOOKUP BUFFER  LENGTH
         DS    XL02            Padding
LBDDNAME DS   0CL08            FILE    DDNAME/ID
LBFILEID DS    FL04            FILE    ID
         DS    CL04            RIGHT   HALF   OF  DDNAME
LBLRID   DS    FL04            LOGICAL RECORD     ID
LBPATHID DS    FL04            LOOK-UP PATH       ID
LBPARENT DS    AL04            PARENT  JOIN   LOOK-UP    BUFFER  ADDR
LBKEYOFF DS    HL02            LOGICAL RECORD KEY OFFSET
LBKEYLEN DS    HL02            LOGICAL RECORD KEY LENGTH
LBSUBNAM DS   0CL08            CALLED  SUBROUTINE NAME
LBTBLBEG DS    FDL08           MEMORY  RESIDENT   TABLE  BEGIN ADDRESS
LBSUBADR DS   0AL04            CALLED  SUBROUTINE ADDRESS
LBTBLEND DS    FDL08           MEMORY  RESIDENT   TABLE  END   ADDRESS
LBSUBWRK DS   0AL04            CALLED  SUBROUTINE WORK   AREA  ANCHOR
LBMIDDLE DS    FDL08           ADDRESS OF MIDDLE  ROW
LBLSTFND DS    FDL08           LAST    CALL  ENTRY  FOUND  ADDRESS
LBLSTRC  DS    FL04            LAST    CALL  RETURN CODE
LBRECCNT DS    FL04            MEMORY  RESIDENT   TABLE  ROW   COUNT
LBRECLEN DS    HL02            MEMORY  RESIDENT   TABLE  ROW   LENGTH
LBEFFOFF DS    HL02            OFFSET  OF EFFECTIVE DATE
LBLKSTK# DS    HL02            LOOK-UP STACK ENTRY  COUNT
LBFLAGS  DS    XL02            PROCESSING FLAGS
LBLSTCNT DS    xl08            LAST CALL EVENT RECORD NUMBER (now bin)
LBINDEX  DS    CL1             Reference data index type  (B / H)
LBHASHT  DS    CL1             hash type (P = pack key before hash)
LBHPRIME DS    FD
LBHASHBEG DS   FD              start of hash table index
LBHASHEND DS   FD              End of hash table index
LBCOLLISION DS  F               Number of hash table collisions
         ORG   LBINDEX
LBSTRTUP DS    CL32            STARTUP  PARAMETERS      (exit lookups)
LBPARML  DS   0D               LOOK-UP  EXIT  PARAMETER    LIST
LBENVA   DS    A               ENVIRONMENT DATA ADDRESS
LBFILEA  DS    A               EVENT FILE  INFO ADDRESS
LBSTARTA DS    A               START-UP    DATA ADDRESS
LBRECA   DS    A               EVENT     RECORD POINTER  - CURRENT
LBEXTRA  DS    A               EXTRACT   RECORD ADDRESS  - CURRENT
LBKEYA   DS    A               LOOK-UP   KEY    ADDRESS
LBANCHA  DS    A               WORKAREA  ANCHOR POINTER    ADDRESS
LBRTNCA  DS    A               RETURN    CODE   ADDRESS
LBRPTRA  DS    A               RESULT    RECORD POINTER    ADDRESS
LBBLKSIA DS    A               BLOCKSIZE
LBPRMLEN EQU   *-LBPARML
LBFNDCA  DS    A               LOOK-UP       FOUND COUNTER ADDRESS
LBNOTCA  DS    A               LOOK-UP NOT   FOUND COUNTER ADDRESS
lbfndcnt DS    fd              LOOK-UP       FOUND COUNTER
lbnotcnt DS    fd              LOOK-UP NOT   FOUND COUNTER
LBSUBENT DS    A               TRUE SUBROUTINE ENTRY POINT
LBWEPATH DS    FL04            Path ID from the workbench
LBEVENTA DS    FD              Address of source/event record
LBKEY    DS   0CL01            LOGICAL RECORD KEY
LBDATA   DS   0CL01            LOGICAL RECORD DATA (OPTIONAL)
LBPREFLN EQU   *-LKUPBUFR      RECORD  BUFFER AREA PREFIX LENGTH
```

Note the overlays: `LBSUBNAM`/`LBSUBADR`/`LBSUBWRK` share storage with
`LBTBLBEG`/`LBTBLEND` (a buffer is either memory resident or exit driven), and
the hash fields (`LBINDEX` onward) share storage with the exit start-up
parameters and parameter list. `LBPARML` is deliberately laid out like the
first part of `GENPARM` so a lookup exit sees the standard parameter list.

`LBFLAGS` bits:

```asm
LBMEMRES EQU   X'80'   MEMORY RESIDENT TABLE
LB64BRES EQU   X'40'   64 BIT MEM RESIDENT TABLE
lbasmtyp EQU   X'20'   CONTINUED DATASPACE TBL
LBEFFDAT EQU   X'10'   EFFECTIVE DATES PRESENT
LBSUBPGM EQU   X'08'   SUBROUTINE CALL (lookup exit)
LBINIT   EQU   X'04'   INITIALIZATION CALLED
LBCLOSE  EQU   X'02'   CLOSE PHASE CALLED
LBEFFEND EQU   X'01'   END DATES PRESENT
* second byte (LBFLAGS+1)
LBV3RED  EQU   X'80'   REF DATA FROM V3 FMT RED
LBADJPOS EQU   X'40'   ADJ STRT POS (V4REH+V3RED)
LBWRTX   EQU   X'20'   TOKEN WRITTEN BY EXIT
LBEXITOPT EQU  X'10'   LU exit is optimizable
```

### 3.2 Table entry prefix (`LKUPTBL`)

Every reference row is stored as a 16-byte prefix followed by the raw record.
The prefix is interpreted differently depending on the index type:

```asm
LKUPTBL  DSECT                 LOOK-UP TABLE  ENTRY DEFINITION
* Field names for Binary search tree
LKLOWENT DS    FDL08           LOW  VALUE ROW ADDRESS
LKHIENT  DS    FDL08           HIGH VALUE ROW ADDRESS
* Fields names for HASH table indexing
         org   LKUPTBL
LKSCOUNT DS    FDL08           Number in this syn chain (collisions)
LKSNEXT  DS    FDL08           Next in synonym chain
LKUPDATA DS   0CL01
LKPREFLN EQU   *-LKUPTBL       LOOK-UP TABLE  ENTRY PREFIX LENGTH
```

```text
 Binary index                         Hash index
 +----------+----------+---------+    +----------+----------+---------+
 | LKLOWENT | LKHIENT  | row ... |    | LKSCOUNT | LKSNEXT  | row ... |
 +----------+----------+---------+    +----------+----------+---------+
   64-bit addresses of children          synonym chain count / next
```

### 3.3 Sizing and the 64-bit pool

After all headers are read, `LOADLKUP` sums `(TBRECLEN + LKPREFLN) *
TBRECCNT` for every file (kept in `TBREFMEM`). If hashing is selected for a
file it also adds `8 * TBHPRIME` bytes for the slot array. The grand total is
rounded up to a megabyte boundary and obtained once with
`IARV64 REQUEST=GETSTOR`; `Refpoolb` (base) and `Refpoolc` (current free
address) carve tables out of it sequentially, so all reference data lives
above the bar (`LB64BRES`).

## 4. Loading the rows (`FILL*` in GVBMR96)

For each header with a positive `TBRECCNT`:

1. Build the 8-byte buffer id `X'FFFFFFFF' || TBFILEID` + `TBRECID` and find
   or allocate the matching `LKUPBUFR` for the logic-table rows that reference
   this file/LR.
2. Open `GREFnnn` through `GVBUR20` (function code 0 = open) and read rows.
3. Apply effective-date options (section 6) and copy `TBINDEX`/`TBHASHT`/
   `TBHPRIME` into `LBINDEX`/`LBHASHT`/`LBHPRIME`.
4. For binary indexes, verify the key of each row is strictly ascending
   compared with the previous row (`FILLKYCK`, an executed `CLC`); a
   non-ascending key ends the run (`FILLSERR`). Hash indexes skip this check.
5. Copy the row into the pool after a zeroed `LKUPTBL` prefix:

```asm
FILLCOPY DS    0H
         XC    LKLOWENT,LKLOWENT  ZERO BINARY SEARCH PATHS
         XC    LKHIENT,LKHIENT
         LGR   R3,R4              LOAD RECORD LENGTH
         LGR   R15,R4
         LA    R2,LKUPDATA
         MVCL  R2,R14             NOTE: R2 IS ADVANCED BY "MVCL"
         LGF   R15,LBRECCNT       INCREMENT   RECORD   COUNT
         LA    R15,1(,R15)
         ST    R15,LBRECCNT
         BRCTG R7,FILLLOOP
```

6. Confirm EOF, then build the index: `BLDPATH` for `LBBINARY`, `BLDHASH` for
   `LBHASH`.

### 4.1 Building the balanced binary search paths (`BLDPATH`)

Because the rows are already sorted, the tree is built in place: the middle
row becomes the root (`LBMIDDLE`) and the routine walks subscripts
top-to-bottom, left-to-right, using `BINSTACK` as an explicit stack to fill
`LKLOWENT`/`LKHIENT` with child addresses. Leaf nodes get zero pointers.

```asm
BLDPATH  ds    0h
         LGHI  R3,1               INITIALIZE  BOTTOM SUBSCRIPT
         LGF   R4,LBRECCNT        INITIALIZE     TOP SUBSCRIPT
         LTGR  R4,R4              ANY   RECORDS   IN TABLE ???
         JNP   BLDEXIT
         LAY   R6,BINSTACK        INITIALIZE CURRENT STACK POINTER
         LGH   R7,LBRECLEN        LOAD TABLE ENTRY  LENGTH
         AGHI  R7,LKPREFLN
         LA    R14,0(R3,R4)       COMPUTE MIDDLE NODE SUBSCRIPT
         SRLG  R14,R14,1
         ...
         STG   R2,LBMIDDLE        SAVE    MIDDLE NODE ADDRESS
```

```text
   sorted rows:   1   2   3   4   5   6   7
                              ^
                           LBMIDDLE (root)
                        /             \
                    2 (1,3)         6 (5,7)
   each LKUPTBL prefix stores LKLOWENT = left child, LKHIENT = right child
```

### 4.2 Building the hash index (`BLDHASH`)

Hashing is opt-in per reference table through the `HASH_TABLE_LU` parameter
(`HASH_TABLE_LU = (LF,LR, mult, PACK/NOPACK)`), with hidden `HASH_MULT`
(default 3) and `DISPLAY_HASH` parameters. `HASHPARM` looks the file/LR up in
the parameter list; when found and the table has no effective dates
(`TBEFFDAT = X'00'`), the header is marked `TBINDEX=TBHASH`, `TBHMULT` and
`TBHASHT` are copied, and `FindPrime` returns the slot count: the record count
is multiplied by `EXEC_HASHMULTB`, raised to a minimum of 1024, made odd, and
advanced to the next prime by trial division up to the square root (the source
notes as a future enhancement the POP recommendation that a `CKSM`-based hash
should avoid primes close to a power of two).

`BLDHASH` carves `8 * LBHPRIME` bytes from `Refpoolc` as `LBHASHBEG..LBHASHEND`
and, for every row:

1. Optionally packs the key 16 bytes at a time into `pgmwork` (`LBHASHT=C'P'`),
   which the source comments describe as a way to reduce collisions for
   numeric keys.
2. Computes `CKSM` over the (packed) key and takes the remainder modulo
   `LBHPRIME` as the slot number.
3. Stores the row address in the slot, or if the slot is occupied, appends the
   row to the synonym chain (`LKSNEXT`), increments the chain head's
   `LKSCOUNT`, counts `LBCOLLISION`, and tracks the collision high-water mark.

```asm
hashloop   cksm r3,r14            Compute Checksum
           d   r2,LBHPRIME+4      remainder in R2 is the hash value
           sllg R2,r2,3           CONVERT SUBSCRIPT  TO SLOT ADDRESS
           if ltg,R2,0(,R3),z     Is this slot empty?
             XC LKSNEXT,LKSNEXT   NO SYNONYMS (NEXT)
             XC LKSCOUNT,LKSCOUNT NO SYNONYMS (COUNTER)
           else   ,               Slot not empty - collision - chain
             ASI LBCOLLISION,1
             do until=(ltg,r2,syn.LKSNEXT,z) loop to end of syn chain
             STG r6,syn.LKSNEXT
             AGSI syncount.LKSCOUNT,1
```

With `DISPLAY_HASH=Y` a hash report file is opened and the per-slot statistics
are written.

## 5. Lookup buffer allocation in PASS1 (`P1LBADR*`)

During PASS1 (see [06-generated-code.md](06-generated-code.md)) each `LU*`
row that does not yet have a buffer for the current thread gets one from
`ALLOLKUP`. The shared portion (`LBKEYOFF` onward) is copied from the template
buffer built during `LOADLKUP`, then the thread-specific fields are set:

```asm
         LH    R0,LKUPSTK#        COPY CURRENT STACK  ENTRY    COUNT
         AHI   R0,1
         STH   R0,LBLKSTK#
         MVC   LBWEPATH,ltluwpath copy workbench path id
         LA    R0,env_area        ENVIRONMENT DATA   ADDRESS
         ST    R0,LBENVA
         LA    R0,LBSTRTUP        START-UP    DATA   ADDRESS
         ST    R0,LBSTARTA
         LA    R0,LBEVENTA
         ST    R0,LBRECA
         ...
         LA    R0,LKUPKEY         LOOK-UP     KEY    ADDRESS
         ST    R0,LBKEYA
         LA    R0,LBLSTRC         RETURN      CODE   ADDRESS
         ST    R0,LBRTNCA
         LA    R0,LBLSTFND        RETURN      RECORD POINTER   ADDRESS
         ST    R0,LBRPTRA
         ...
         OI    LBNOTCA,X'80'      MARK END-OF-PARAMETER-LIST
P1LBADR4 ST    R5,LTLBADDR        SAVE BUFFER ADDRESS (CURR THREAD)
         oi    ltflag1,ltlkup_alloc    and mark the lt entry
```

A `JOIN` row (or the first step of a path when `SVLUJOIN` is still zero) makes
this buffer the parent for subsequent steps:

```asm
         if CLC,LTFUNC,eq,JOIN,or, JOIN                                +
               OC,SVLUJOIN,SVLUJOIN,z Parent addr 0 (prev LKLR step 1)
           ST    R5,SVLUJOIN       SAVE PARENT JOIN/path ADDRESS
           ST    R5,LBPARENT       SAVE PARENT JOIN ADDRESS (THIS path)
         endif
```

For memory-resident tables PASS1 then adjusts the source field offsets of
`SETL`/`ADDL`/`SUBL`/`MULL`/`DIVL`/`CFLA` and other `*L` rows so they address
the record inside the `LKUPTBL` entry (Version 4 "adjust positions").

## 6. Effective dates

`TBEFFDAT` in the header (Version 3 `C'Y'` is remapped to `X'02'`) selects
the date handling. `LOADLKUP` extends the key so the 4-byte date is part of
the compared key and remembers where it starts:

```asm
         LH    R0,TBKEYLEN        LOAD  KEY  LENGTH (EXCL DATE)
         STH   R0,LBEFFOFF
         AHI   R0,4-1             ADD    EFFECTIVE   DATE LENGTH
         STH   R0,LBKEYLEN        UPDATE KEY LENGTH (INCL DATE)  (-1)
         CHI   R15,ENDRANGE       END   DATES   PRESENT   ???
         JNE   FILLENDO
         OI    LBFLAGS,LBEFFEND
         J     FILLEFFD
FILLENDO CHI   R15,ENDONLY        END   DATES   PRESENT   ???
         JNE   FILLEFFD
         OI    LBFLAGS,LBEFFEND
         J     FILLCNT
FILLEFFD OI    LBFLAGS,LBEFFDAT   SET   EFFECTIVE DATE INDICATOR
```

| `TBEFFDAT` | Meaning | Flags set |
|------------|---------|-----------|
| `X'00'` | no dates; exact key match (hash index allowed) | none |
| `X'01'` | start dates only | `LBEFFDAT` |
| `ENDRANGE` (`2`) | start and end dates | `LBEFFDAT + LBEFFEND` |
| `ENDONLY` (`3`) | end dates only | `LBEFFEND` |

Any non-zero option forces `LBBINARY` (the sizing loop only considers hashing
when `TBEFFDAT = X'00'`), because the search must fall back to the nearest
neighbouring row. `LBKEYLEN` is stored as *length minus one* ready for `EX`.

## 7. Searching from generated code

### 7.1 Key assembly

Lookup keys are built in the thread's 256-byte `LKUPKEY` work area
(`GVBMR95W`). The first four bytes hold the logical-record id; field values
follow. Function codes that populate it:

| Function | Model code | Behaviour |
|----------|------------|-----------|
| `LKLR` | `MDLLK01` | `MVC LKUPKEY(4),lrid` from the literal pool (`CSLRID`) |
| `JOIN` | `MDLLK02` | optimized `LKLR`: if `LBLSTCNT` equals the current event record count (`GPRECCNT`) the previous result is reused - branch FALSE if `LBLSTFND` is `X'FF..'`, TRUE otherwise; else store the count and move `LBLRID` |
| `JOIN` (token source) | `MDLlk03` | same optimization keyed on the token `RE` row's record counter (`ltfilcnt` offset in the literal pool) |
| `LKC`, `LKS`, `LKDC` | `MDLDT34` | constant / symbol / date constant to key |
| `LKE`, `LKL`, `LKP` | `DT*` format arrays | event / lookup / previous-record field to key (record type 4 is promoted to 5 during load so the full source descriptor is available) |
| `LKDE`, `LKDL` | `MDLDT01/09/15/20` | date field from event / lookup record |
| `LKDA` | `MDLDT50` | date accumulator |
| `KSLK` | `MDLKS01` | `MVC EXTREC+0(0),LKUPKEY` - save the key into the extract record title-key area (patched by `CSTTLOFF` in PASS2) |

For `LKE` rows `GVBMR96` calls `INCKYPOS` to advance the running key position
by the field length, so successive `LK*` rows append to the key.

### 7.2 `LUSM` - search memory

```asm
MDLLUSM  llgt  R5,0(,R2)          LOAD   LOOKUP BUFFER ADDRESS
MDLLUSMX LLGT  R15,0(,R12)        ADDRESS SEARCH ROUTINE without hi bit
         BASR  R10,r15            SEARCH ROUTINE
MDLLUSMF jlu   *+l'*              "RECORD NOT FOUND" BRANCH (FALSE)
MDLLUSMT jlu   *+l'*              "RECORD     FOUND" BRANCH (OPTIONAL)
```

`CSSRCHR` is relocated to the routine chosen for this buffer:

* `LBINDEX = LBBINARY` - entry `SnnnTBLA` from `GVBSRCHR`, where `nnn` is the
  key length including any date. `SRCHADDT` in `GVBMR95` holds 256 V-cons
  generated by `GVBSRCH KEYLEN=n,VCON=Y`, so the search is a fixed-length
  `CLC` with no length register.
* `LBINDEX = LBHASH` - `HASHLKUP` inside `GVBMR95`.

The routine returns to `R10` for *not found* and to `R10 + l'mc_jump` (past
the FALSE branch) for *found*, leaving R5 pointing at the `LKUPBUFR`; the
`lkupcode` prefix (`lg R5,LBLSTFND-LKUPBUFR(,R5)`) then turns R5 into the
address of the found row for the following `*L` functions.

#### Binary search (`GVBSRCH` macro, expanded per key length)

```asm
&SLAB.TBLA DS  0D
         ltg   R4,LBLSTFND        CHECK IF THIS KEY SAME AS LAST ???
         JNP   &SLAB.INIT
         lgr   R14,r4
         CLC   LKUPKEY+4(&KEYLEN),0(R14) BYPASS SEARCH (USE PREVS) ?
         be    l'mc_jump(,r10)    RETURN  TO  FOUND ADDRESS
&SLAB.INIT LG  R4,LBMIDDLE        LOAD ADDRESS  OF MIDDLE ENTRY
&SLAB.LOOP lgr R3,R4              SAVE    LAST  ENTRY EXAMINED
         LA    R14,LKUPDATA
         CLC   LKUPKEY+4(&KEYLEN),0(R14)
         JL    &SLAB.TOP            LOWER TOP
         JH    &SLAB.BOT            RAISE BOTTOM
&SLAB.FND LA   R14,LKUPDATA       LOAD ADDRESS  OF DATA
         stg   R14,LBLSTFND       SAVE ADDRESS  IN BUFFER PREFIX
         agsi  lbfndcnt,bin1       INCREMENT   COUNT
         b     l'mc_jump(,r10)    RETURN  TO  FOUND ADDRESS
&SLAB.TOP LTG  R4,LKLOWENT        LOAD ADDRESS   OF LOWER  VALUE NODE
         JP    &SLAB.LOOP
         J     &SLAB.CHK          NO  - CHECK FOR EFFECTIVE DATE
&SLAB.BOT LTG  R4,LKHIENT         LOAD ADDRESS   OF HIGHER VALUE NODE
         JP    &SLAB.LOOP
```

Key points:

* The comparison starts at `LKUPKEY+4`, i.e. the LR id is not part of the
  stored key.
* A hit on the previously found row (`LBLSTFND`) short-circuits the search.
* Not found: `LBLSTFND` is set to -1 and, if `LBPARENT` is positive, the
  parent join buffer's `LBLSTFND` is also set to -1 so a later `JOIN` row
  resolves immediately to FALSE for the same event record.
* For key lengths of five or more the macro adds effective-date logic. When
  `LBEFFDAT` is on and the key carries a non-zero date, the last examined node
  (`R3`) is re-examined: if the key compared high the node itself is the
  candidate, otherwise the routine steps back one physical row (rows are
  contiguous, `LBRECLEN + LKPREFLN` apart) and checks that the root key
  excluding the 4-byte date still matches. With `LBEFFEND` also on, the
  requested date must not exceed the end date stored immediately after the key
  (`LKUPDATA + LBKEYLEN + 1`). `ENDONLY` tables step forward instead
  (`&SLAB.END/&SLAB.ENDH/&SLAB.ENDL`).

```text
   key without date       key with start date (LBEFFDAT)
   exact match only       find greatest row whose (root key == request)
                          and start date <= request date
                          optionally: request date <= end date (LBEFFEND)
```

#### Hash search (`HASHLKUP` in GVBMR95)

```asm
HASHLKUP DS    0D
         lgh   R15,LBKEYLEN       LOAD  KEY  LENGTH (-1)
         ltg   R14,LBLSTFND       CHECK IF THIS KEY SAME AS LAST ???
         JNP   HASHINIT
         exrl  R15,HASHSAME
         be    l'mc_jump(,r10)    RETURN  TO  FOUND ADDRESS
HASHINIT DS    0h
         la    r14,LKUPKEY+4
         if cli,lbHASHT,EQ,C'P'   ... pack key into pgmwork ...
         ...
hashlp   cksm  r1,r14              Compute Checksum
         jnz   hashlp
         d     r0,LBHPRIME+4
         sllg  R1,r0,3             CONVERT SUBSCRIPT  TO SLOT ADDRESS
         AG    R1,LBHASHBEG
         if ltg,R4,0(,R1),z        Is this slot empty?
           J  HASHNOT
         else
           do until=(ltg,r4,LKSNEXT,z) loop to end of syn chain
             LA    R14,LKUPDATA
             exrl  R3,HASHSAME     Matching key?
             je    HASHFND
           enddo
         endif
HASHNOT  ds    0h
```

The same pack/CKSM/modulo sequence used at build time is repeated at run time,
then the synonym chain is walked with an executed `CLC` (`HASHSAME`).

### 7.3 `LUEX` - lookup exit

```asm
MDLLUEX  llgt  R5,0(,R2)                  LOAD LOOKUP  BUFFER  ADDRESS
         lay   r15,PARM_AREA
         MVC   LBPARML-LKUPBUFR(8,R5),0(r15)
         STG   R6,LBEVENTA-LKUPBUFR(,r5)  Save Event record addr in LB
         ST    R7,LBEXTRA-LKUPBUFR(,R5)   PASS EXTRACT RECORD  ADDR
         LA    R0,LKUPKEY                 PASS KEY     ADDRESS
         ST    R0,LBKEYA-LKUPBUFR(,R5)
         ...
         LA    R1,LBPARML-LKUPBUFR(,R5)   LOAD PARAMETER  LIST  ADDR
         llgf  R15,LBSUBADR-LKUPBUFR(,R5) LOAD SUBROUTINE ADDR
         BASsm R14,R15                    CALL SUBROUTINE
         if    (oc,gp_error_reason,gp_error_reason,nz)
           MVC   ERRDATA(8),LBSUBNAM-LKUPBUFR(R5)
           bas  r9,errwto
         endif
         lt    R15,LBLSTRC-LKUPBUFR(,R5)  SUCCESSFUL  ???
         jnz   mdlluexe
         agsi  lbfndcnt-LKUPBUFR(r5),bin1
         LH    R0,LBLKSTK#-LKUPBUFR(,R5)  SAVE STACK    COUNT
         ST    R0,gpjstpct
         lg    R15,LBLSTFND-LKUPBUFR(,R5) SAVE POINTER  IN STACK
         llgt  R14,gpjstka
MDLLUEXS AHI   R14,0
         MVC   0(4,R14),LBLRID-LKUPBUFR(R5)
         stg   R15,4(,R14)
MDLLUEXT jlu   *+l'*                 "Record found" branch true
```

The exit receives the `LBPARML` list (environment, file, start-up, event
record, extract record, key, anchor, return code, result pointer). Return
codes in `LBLSTRC`:

| RC | Effect |
|----|--------|
| 0 | found: `LBLSTFND` points at the result, join stack entry (`GPJSTKA` + `CSLKPSTK` offset) receives LR id and address, TRUE branch |
| 4 | not found: FALSE branch; parent `LBLSTFND` set to `X'FF..'` |
| 8 | skip this event record (`EVNTPREV`) |
| 12 | disable the view (`DISABREQ_model` through the view's `NV` row) |
| 16 (other) | abort the run (`ABORTEX_indirect`) with the exit name in `ERRDATA` |

A non-zero `gp_error_reason` returned by the exit produces a WTO before the
return code is examined.

## 8. Multi-step paths and the join stack

A lookup path with several steps is emitted as a chain of `LK*`/`LUSM` groups.
`LBPARENT` links each step's buffer to the path's parent buffer (set from
`SVLUJOIN` in PASS1). At run time:

* `JOIN` reuses the parent's outcome for the current event record through
  `LBLSTCNT`/`GPRECCNT` and `LBLSTFND` (`MDLLK02`), so a path whose first step
  already failed does not re-search.
* `LBLKSTK#` numbers the step; `GPJSTPCT`/`GPJSTKA` in the `GVBX95PA`
  parameter area expose the current join step count and stack to exits.
* `LTLUBOFF`/`ltlkup_offset` and the `lkupcode` prefix let any following `*L`
  function address the found row of the *correct* step.

```text
  event record --LKE--> LKUPKEY --LUSM--> buffer A (step 1) --found-->
     LKL(from A) --> LKUPKEY --LUSM--> buffer B (step 2, LBPARENT=A) --found-->
     SETL / CFLA / SKL ... use LBLSTFND of B via lkupcode prefix
```

## 9. Statistics and reporting

* `lbfndcnt`/`lbnotcnt` are incremented on every found/not-found result and
  rolled into `LPLKPFND`/`LPLKPNOT` in the literal pool header for the
  control report.
* `GVBMR96` prints one control-report line per reference file (`tracref`
  format with file id, LR id and record count) under `MSGLVL_DEBUG`, and the
  reference-phase alias records `iref_rcnt`/`iref_bcnt` (reference records and
  bytes read) for the "Reference files read / records read / bytes read"
  summary lines.
* `GVBMR87` prints the `IREF` report while reading `REFRRTH` (DDNAME, key
  length, record count, memory) and totals `refrect`/`refmemt`.

## 10. Format-phase lookups (`GVBMR87`)

`GVBMR87 FILLLKUP` performs the same job for `GVBMR88`: it opens `REFRRTH`
(`HDRFILE DCB ... DDNAME=REFRRTH ... EODAD=FILLEOF`), builds `REFRnnn` from
`TBXFILE#`, loads each table with a `LKUPTBL` prefix per row and builds the
binary search paths used by sort-title lookups (`KSLK` keys saved in the
extract record, resolved during report formatting). The format phase does not
use the hash index. `GVBMR88C.mac` carries its own copy of the effective-date
constants (`ENDRANGE EQU 02`).

## 11. Error codes

| Constant (`GVBUTEQU`) | Number | Condition |
|-----------------------|--------|-----------|
| `REH_FILE_DUPLICATE` | 108 | duplicate reference table header |
| `REH_FILE_DATA_ERR` | 137 | Version 3 (`$$`) header or invalid REH data |
| `REH_COUNT_ERR` | 164 | REH record count differs from VDP 0650 GREF count |
| `OPEN_REH_FAIL` | 106 | `EXTRREH`/`REFRREH` open failed |
| `REF_READ_ERR` | 124 | `GVBUR20` returned an unexpected code while reading `GREFnnn` |
| `LKUPBUFR_LF_NOT_FOUND` | 120 | PASS1 could not find a lookup buffer for a logic-table row's file |
| `FILLSERR` path | - | keys not in ascending sequence for a binary-indexed table |

See [13-parameters-ddnames-messages.md](13-parameters-ddnames-messages.md) for
the full message table.
