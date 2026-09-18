# 06 - Generated Machine Code

The Performance Engine does not *interpret* the logic table at run time.
During initialisation it **assembles a program**: for every logic-table row
it copies a hand-written HLASM skeleton ("model code") out of `GVBMR95`
into a per-thread code buffer, patches operand lengths, offsets and branch
displacements into that copy, and builds a literal pool alongside it. The
result is straight-line System z machine code that runs once per event
record with no dispatch overhead. This chapter explains that mechanism.

Sources: `ASM/GVBMR95.asm` (function table, model code, relocation tables,
`PASS2`), `ASM/GVBMR96.asm` (`PASS1`, `ALLOCLIT`, `LKUPPREF`),
`MAC/GVBMR95C.mac` (`FUNCTBL`, `NVPROLOG`, `LITP_HDR`, `CS*` codes),
`MAC/LKUPCODE.mac`.

## 6.1 The big picture

```text
 GVBMR95.asm (static)                      per-thread buffers (dynamic)

 +----------------------+                 +-------------------------------+
 | FUNCTBL  (~75 major  |                 |  code segment  (LTCODSEG …)   |
 |  functions, each     |   PASS1 copies  |  +-------------------------+  |
 |  -> one or many      | --------------> |  | NV prologue             |  |
 |  model-code entries) |                 |  | [trace BAS]  row 1 code |  |
 +----------+-----------+                 |  | [LKUPPREF]   row 2 code |  |
            |                             |  | ...                     |  |
            v                             |  | ES epilogue -> EVNTPREV |  |
 +----------------------+                 |  +-------------------------+  |
 | MDLxxxx  model code  |                 +-------------------------------+
 | MDLxxxxL length      |                                  ^
 | MDLxxxxP litpool use |                                  | R2 = literal pool
 | MDLxxxxR relocation  |                                  |      + 512K
 |          table       |                 +-------------------------------+
 +----------------------+                 |  literal pool                 |
                                          |  +-------------------------+  |
   relocation entry:                      |  | LITP_HDR (per view)     |  |
   AL1(CS* code), AL1(offset in model)    |  | constants, addresses,   |  |
   ...  X'FF',X'FF'                       |  | acc. values, masks ...  |  |
                                          |  +-------------------------+  |
                                          +-------------------------------+
```

Two passes patch the copied skeletons:

* **PASS1** (`GVBMR96`) - copies model code and literals, and resolves
  every relocation whose value is known before all rows have been placed
  (field lengths/offsets, literal addresses, lookup buffers, ...).
* **PASS2** (`GVBMR95`, after `GVBMR96` returns) - resolves the
  relocations that need the final position of *other* rows: TRUE/FALSE
  branch displacements and title-key offsets, and finalises NV/ES
  prologue/epilogue instructions.

## 6.2 The function table (`FUNCTBL`)

Every function code the logic table can contain is described by a
`FUNCTBL` entry (`MAC/GVBMR95C.mac`):

```asm
FUNCTBL  DSECT
FCFUNC   DS    CL04        function code, e.g. 'CFEC'
FC_RTYP  DS    XL01        operand record-type qualifier
FCCODELN DS    HL02        length of model code
FCLITPLN DS    HL02        literal pool bytes used (excluding field data)
FCMODELA DS    AL04        A(model code)
FCRELOCA DS    AL04        A(relocation table)
FCLTSUB  DS    HL02        literal pool subroutine index
FCP1SUB  DS    HL02        PASS1 subroutine index
FCP2SUB  DS    HL02        PASS2 subroutine index
FCENTLEN EQU   *-FUNCTBL
```

A typical entry in `GVBMR95.asm` is:

```asm
CF_EE_01 DC    CL4'CFEE',AL1(FC_RTYP05)                COMPARE
         DC    AL2(MDLCFEEL),AL2(MDLCFEEP)
         DC    AL4(MDLCFEE),AL4(MDLCFEER)
         ...
```

`FC_RTYP` records which **external logic-table record layout** the row
was read with (`FC_RTYP01` header, `02` new view, `03`/`04`/`05` formats
0/1/2, `06` read event, `07` write extract, `08` compare constant, `09`
variable name, `0A` name/value, `0B` calculation, `0C`/`0D` variable
function formats 1/2 - see [05-logic-table.md](05-logic-table.md)). It
tells PASS1 which `LTREDEFN` fields are valid for the row.

### 6.2.1 Major-function ordering

The **major function table** in `GVBMR95.asm` lists the ~75 four-character
function codes in a fixed order (`ADDA ADDC ADD CFA CFAA ... FN`).
`GVBMR96`'s `LTBLLOAD` looks the row's `LTF1_FUNCTION_CODE` up in this
table and stores the resulting **sequence index** in the row so that the
selection structure for that function can be found without another
compare chain. The index is then used against one of four shapes:

| shape | when used | example |
|---|---|---|
| single entry | one skeleton for the function | `GOTO`, `NOOP`, `WRXT`, `KSLK` |
| 1-D vector by source format | target format is fixed (e.g. the target is always a constant or accumulator) | `DTA` (`dta_ss`), `CTA`, `SETA` |
| 13x13 array by target/source format | arithmetic/compare/set between two typed operands | `CFEC`, `SETE`, `ADDE`, `SKE` |
| 4 signed/unsigned 13x13 arrays (UU, US, SU, SS) | sign of both operands matters | `dt_array` (`DTE`), `cfxx_sortp_*` |

The arrays are built with `ORG` statements over a zero-filled block
(`dc (array_size)f'0'` then `org dt_array+(fc_num)*row_length` ...). The
first word of each array is the **default entry**; `array_select` in
`GVBMR96` indexes the array and falls back to that default when the
selected cell is zero:

```asm
array_select   llgt r15,0(,r5)             default entry
               if cli,ltcolsgn,eq,C'Y'     signed target -> arrays 3/4
                 aghi r5,array_size*2
               endif
               if cli,ltsign,eq,C'Y'       signed source -> arrays 2/4
                 aghi r5,array_size
               endif
               ic   r0,ltfldfmt+1 ; sllg r0,r0,2        source col * 4
               ic   r6,ltcolfmt+1 ; ms   r6,=a(row_length)  target row
               agr  r6,r0
               if ltgf,r5,0(r6,r5),z
                 lgr r5,r15                zero cell -> use default
               endif
```

The 13 format positions are `0 default, 1 ALNUM, 2 ALPHA, 3 NUM, 4 PACK,
5 SORTP, 6 BIN, 7 SORTB, 8 BCD, 9 MASK, 10 EDIT, 11 FLOAT, 12 GEN#`.
For binary (`FC_BIN`/`FC_SORTB`) operands the selected cell points at the
"bin124/bin124" `FUNCTBL` entry and the function table places the
bin124/bin8, bin8/bin124 and bin8/bin8 variants immediately after it;
`array_select` skips `FCENTLEN`-sized entries according to whether the
source and/or target length is 8 bytes (`ltfldlen+1 >= 7`). The default
entry returned in R15 is also kept (`default_func`) for rows with content
codes (dates), which need the generic skeleton. Long fields > 256 bytes use
skeletons relocated with `CSTGTLNE`, `CSSRCLNE`, `CSLOOPC` and `CSSRCRM`.

## 6.3 Model code skeletons

Each skeleton is written in HLASM inside `GVBMR95` under the same
`USING`s the generated code will run with:

```asm
         using (thrdarea,thrdend),r13   thread work area
         using genenv,env_area
         using genparm,parm_area
         using genfile,file_area
         USING EXTREC,R7                 extract record being built
         using litp_hdr+524288,r2        literal pool (R2 = pool - 512K)
```

The naming convention is rigid, and PASS1/PASS2 depend on it:

| symbol | meaning |
|---|---|
| `MDLxxxx` | first instruction of the skeleton |
| `MDLxxxxL EQU *-MDLxxxx` | length copied into the code buffer |
| `MDLxxxxP EQU n` | literal-pool bytes the row needs *in addition to* field data |
| `MDLxxxxR DC AL1(CS…),AL1(offset) … DC 2XL1'FF'` | relocation table |

Example - "write extract record, no exit":

```asm
MDLWRXT  llgt  R5,0(,R2)          LOAD   LOGIC TBL ROW     ADDRESS
         BAS   R10,WRTEXT_Indirect BRANCH AND WRITE EXTRACT RECORD
MDLWRXTL EQU   *-MDLWRXT
MDLWRXTP EQU   4                  LITERAL POOL USAGE (EXCL FIELD LEN)
MDLWRXTR DC    AL1(CSLTROFF),AL1(MDLWRXT+02-MDLWRXT) LOGIC TBL ROW ADDR
         DC   2XL1'FF'
```

The `llgt R5,0(,R2)` has a displacement of zero in the skeleton; PASS1
stores the current row's address in the literal pool and patches the
displacement at offset 2 (`CSLTROFF`).

Example - memory lookup:

```asm
MDLLUSM  llgt  R5,0(,R2)          LOAD   LOOKUP BUFFER ADDRESS
MDLLUSMX LLGT  R15,0(,R12)        ADDRESS SEARCH ROUTINE without hi bit
         BASR  R10,r15            SEARCH ROUTINE
MDLLUSMF jlu   *+l'*              "RECORD NOT FOUND" BRANCH (FALSE)
MDLLUSMT jlu   *+l'*              "RECORD     FOUND" BRANCH (OPTIONAL)
MDLLUSML EQU   *-MDLLUSM
MDLLUSMP EQU   4
MDLLUSMR DC    AL1(CSLBAOFF),AL1(MDLLUSM+2-MDLLUSM)   LOOK-UP  BUFR
         DC    AL1(CSSRCHR),AL1(MDLLUSMX+2-MDLLUSM)  search routine
         DC    AL1(CSFALSEM),AL1(MDLLUSMF+2-MDLLUSM)  FALSE  BRANCH
         DC    AL1(CSTRUEO),AL1(MDLLUSMT+2-MDLLUSM)   TRUE   BRANCH
         DC   2XL1'FF'
```

Generated code that needs a real subroutine (`WRTEXT`, `GVBDL96`, trace,
date arithmetic) calls it through a small **indirect stub** in `GVBMR95`
(`WRTEXT_Indirect`, `call96at` .. `call96x`, `MR95TRAC`, `fnxcsub_indirect`)
so that the skeleton only needs a relative `BAS`, which works from any
code-buffer address.

### 6.3.1 Register contract inside generated code

From the register map at the top of `GVBMR95.asm` and the `USING`s above:

| register | content while generated code runs |
|---|---|
| R13 | `THRDAREA` (thread work area; also save area) |
| R12 | `GVBMR95` base (allows `LLGT R15,0(,R12)`-style access to engine V-cons) |
| R11 | `NVCONST` (view header constants of the current NV) |
| R10 / R9 | subroutine return addresses (1st / 2nd level) |
| R8 | current extract column address (`LAY R8,EXTREC+0` in NV prologue) |
| R7 | extract record (`EXTREC`) address |
| R6 | current event record address |
| R5 | current reference record / lookup buffer address |
| R4, R3 | work; R3 = previous record (`loadprev`) when a `..P` operand is used |
| R2 | literal pool address minus 512K |
| R14, R15, R1, R0 | work |

The 512K bias on R2 exists so that both positive and negative 20-bit
long-displacements (`LLGT`, `LAY`, `STY`) can reach a 1 MB literal window.

## 6.4 Relocation tables and `CS*` substitution codes

A relocation table is a list of `(type, offset)` byte pairs terminated by
`X'FFFF'`. `type` is one of the `CS*` equates in `MAC/GVBMR95C.mac`
("EQUates above - branch tables dependencies in 95/96"), and it indexes a
branch table `P1RELOTB` in `GVBMR96` and a `select` in `GVBMR95`'s PASS2.
Each code says *what value* to compute and *how to store it* at
`code_base + offset`:

| code | # | meaning | resolved in |
|---|---|---|---|
| `CSSRCLN` / `CSSRCLNL` / `CSSRCLNR` | 1-3 | source field length (whole / left half / right half of an `SS` length byte) from `LTFLDLEN` | PASS1 |
| `CSSRCLOF` | 5 | long source field offset (`LTFLDPOS`) | PASS1 |
| `CSTGTLN` / `CSTGTLNL` / `CSTGTLNR` | 6-8 | target field length from `LTCOLLEN` | PASS1 |
| `CSv1len`, `CSV1OFF`, `CSV2OFF` | 9-11 | value 1 length; copy value 1 / value 2 to the literal pool and patch its offset | PASS1 |
| `CSTRUEO` / `CSTRUEM` | 12-13 | TRUE branch displacement (optional / mandatory) | PASS1 (omission), PASS2 (value) |
| `CSFALSEO` / `CSFALSEM` | 14-15 | FALSE branch displacement | PASS1 (omission), PASS2 (value) |
| `CSRELOPR` / `CSRELOP2` | 16-17 | relational operator (`EQ NE LT LE GT GE BW EW` from `LTRELOPR`/`LTVVROPR`) -> branch mask | PASS1 |
| `CSLTROFF` | 18 | address of this logic-table row in literal pool | PASS1 |
| `CSLBAOFF` / `CSLBAOF2` | 19-20 | lookup buffer address (first / second) | PASS1 |
| `CSCOLNO`, `CSCTACUM`, `CSCTACUM_12` | 21, 47, 55 | "CT" column number | PASS1 |
| `CSSRPCT`, `CSSRPSRC`, `CSSFTDIG` | 23, 49, 54 | `SRP` shift amount to align decimal places | PASS1 |
| `CSLSTBYT` | 24 | offset of last/rightmost target byte | PASS1 |
| `CSBYTMSS` / `CSBYTMSK` | 25-26 | `ICM`/`STCM` byte masks for 1-4 byte binaries | PASS1 |
| `CSLRID`, `CSKEYLEN`, `CSLVLOFF`, `CSLKPSTK` | 27, 28, 30, 53 | logical record id, lookup key length, hierarchical lookup level, lookup stack offset | PASS1 |
| `CSTTLOFF` | 29 | title key offset in extract record | PASS2 |
| `CSACCOFF`, `CSACCOF2`, `CSACCVAL`, `CSACCLEN` | 31-34 | accumulator address / second accumulator / constant / length | PASS1 |
| `CSTRTTBL` | 35 | `TRT` table for numeric class test (signed/unsigned packed, zoned) | PASS1 |
| `CSSUBCTR`, `CSSUBLEN`, `CSSUBOFF`, `CSSUBVAL`, `CSsubctrcx`, `CSsubctrxc` | 36-39, 41, 43 | substring-compare loop parameters | PASS1 |
| `CSSDNLN`, `CSSDNSLN`, `CSSRTLEN` | 40, 50, 42 | sort-descending lengths (target/source), sort key length | PASS1 |
| `CSCALLVW` | 44 | "call view" machine code (write to a VDP-200 partition / token) | PASS1 |
| `CSJUSOFF`, `CSTGTLOF` | 46, 52 | justified target offset, long target offset | PASS1 |
| `CSdl96call`, `CSdl96callr`, `CSdl96calln`, `CSdl96calln_r` | 56-58, 67 | choose the `call96*` stub for a `GVBDL96` conversion | PASS1 |
| `csdfpexp`, `csdfpopt`, `csdfpexps` | 60-62 | decimal-floating-point exponent / FP register load optimisation | PASS1 |
| `CSV1OFFR`, `CSV2OFFR`, `CSTGTLOFR`, `CSSRCLOFR` | 63-66 | "reversed" operand variants | PASS1 |
| `CSSRCHR` | 68 | address of the binary-search / hash routine for this lookup | PASS1 |
| `CScmpdt1`, `CScmpdt2` | 69-70 | compare setup for `CFxC`/`CFCx` with dates | PASS1 |
| `CSTGTLNE`, `CSSRCLNE`, `CSLOOPC`, `CSSRCRM` | 71-74 | long-field (> 256) lengths, loop counter, remainder | PASS1 |

Because the offsets are one byte, a single skeleton is limited to 256
bytes; larger operations are split into several skeletons or loop inside
the engine's subroutines.

## 6.5 Literal pools

All literal pools live in one contiguous area allocated by `ALLOCLIT`
(`GVBMR96`): `litpcnt` work units x 1 MB (`GETMAIN RU,LOC=(ANY),
BNDRY=PAGE`, preceded by the `POOLEYEB` eyecatcher), tracked by
`LITPOOLB`/`LITPOOLC`/`LITPOOLM`. Each `FUNCTBL` entry carries `FCLITPLN`,
the pool bytes its model code needs, but the sizes that matter at run time
are the actual ones measured while the pool is filled: `LITPSIZE` stores the
double-word-rounded size of each view's pool in `NVLITPSZ` and accumulates
it into `ESPOOLSZ`, which becomes `LTESLPSZ` in the `ES` row. After PASS1
the unused tail of the area is `FREEMAIN`ed and the actual total is kept in
`LITPOOLS`/`LITPool_sz`. Statistics for the "~LITP" section of the control
report are kept in `litptots`/`litpstat` while the pools are being filled.

Every view's section of the pool starts with a `LITP_HDR`
(`MAC/GVBMR95C.mac`):

```asm
LITP_HDR DSECT                 VIEW   LITERAL POOL   HEADER
LPVADDR  DS    AL04            Current view code address
LPlurtnc DS    fL04            Last lookup return code
lp_saved_litp ds  al04         saved litp_hdr (callview)
lp_return_adr ds  al04         return address (callview)
lp_nvcons_adr ds  al04         save for r11 if wrtk/wrtx goes to nv
LPEXTCNT DS    PL06            VIEW EXTRACT  RECORDS   WRITTEN  COUNT
LPLKPFND DS    PL06            VIEW LOOKUPS  FOUND     COUNT
LPLKPNOT DS    PL06            VIEW LOOKUPS  NOT FOUND COUNT
         DS    XL02
lp_prev_litpo  ds al04         offset of previous literal pool
lp_base_litp   ds al04         Always contains address of base lit pool
lp_RE_addr     ds al04         Address of RE entry
lp_r6_save     ds cl08         r6 save area event record address
LITPHDRL EQU   *-LITP_HDR
```

After the header come, in row order, the items PASS1 copies for each row:
row addresses (`SAVEROWA`), lookup-buffer addresses (`SAVELBA`,
`SAVEADDR`), constant values (`COPYVAL1`), accumulator constants
(`COPYACC`), column numbers (`SAVECOL#`), `TRT` tables, search-routine
addresses and so on. The pool therefore doubles as the view's **per-thread
state**: `LPEXTCNT`/`LPLKPFND`/`LPLKPNOT` are the counters the control
report prints per view.

### 6.5.1 Temporary pools and cloned sets

Constants are first gathered into a *temporary* literal pool during
`LTBLLOAD`/`PASS1` and copied into the code buffer's pool once the view's
size is known ("copy previous temporary literal pools into the code
buffer" is PASS1 step 3). When an `ES` set is **cloned** (`LTESCLON`, one
copy per thread reading the same file class), `P1FUNES` does not rebuild
the pool: it advances `LITPOOLC` to the next pool address and re-copies
the same literal images, so each thread has private counters but identical
constants.

Token (`RETK`/`WRTK`) views need a second pool section (`litp_base_tok`)
because the token writer's view code runs inside the reader's ES; the
`lp_saved_litp`/`lp_return_adr`/`lp_nvcons_adr` fields save and restore
R2/R11 across that "call view" (`CSCALLVW`).

## 6.6 View prologue (`NV`) and set epilogue (`ES`)

Every view's code segment starts with the `MR95NV` skeleton, whose layout
is mapped by the `NVPROLOG` DSECT (the assembler `ASSERT`s that
`MDLNVL = NVCODELN`):

```asm
MR95NV   jlnop *                 "NOP" BRANCH INSTR(DISABLE VIEW)
         LAY   R8,EXTREC+0        INITIALIZE COLUMN DATA  POINTER
         BRAS  R14,MDLNVBEG       BRANCH AROUND HEADER (SET R14)
         DC    FL4'0'             VIEW    ID          (NVVIEWID)
         DC    AL4(0)             A(LOGIC TABLE ROW FOR "NV") (NVLOGTBL)
         DC    FL4'0'             LITERAL POOL  SIZE  THIS VIEW (NVLITPSZ)
         DC    AL4(0)             A(CODE FOR NEXT VIEW) (NVNXVIEW)
MDLNVBEG MVC   gpview#,NVVIEWID
         lay   r15,lpvaddr
         mvc   0(l'lpvaddr,r15),nvlogtbl
         la    r11,nvconst        get start of NV constants
         nop   trace              BRANCH TO  TRACE  ROUTINE (NOP)
         LG    R6,RECADDR         Initialize Event Record Base Register
```

* The leading `jlnop` (`NVNOP`) is the **view disable switch**. PASS1 sets
  its displacement to the next view (`NVNXVIEW`, `P1FUNNV4`); PASS2 turns
  the NOP into an unconditional branch (`OI NVNOP+1,X'F0'`) when the view is
  not in the `RUNVIEWS` list, so a disabled view costs one branch per
  event record.
* The `nop trace` is replaced by `BAS R10,MR95TRAC` when `TRACE` is active
  for the view (`P1NVNOP`/`P1TRACE`); PASS2 later skips those 4 bytes when
  computing offsets (`aghi R3,4`).
* PASS2 (`P2FUNNV`) also computes the offset of the first sub-total column
  (`EXSRTKEY + LTSORTLN + LTTITLLN + LTDATALN`) and patches it into the
  `LAY R8` (`NVLAYR8`).

The **`ES` epilogue** (`MDLES`) is simply:

```asm
MDLES    B     EVNTPREV           READ  NEXT EVENT RECORD
         B     EVNTPREV           READ  NEXT EVENT RECORD (DUPLICATE)
```

so that when the last view has been processed for the current record,
control returns to the engine's read loop. For a set with no event file
(`LTFRSTRE` = 0) PASS2 overwrites both words with `BRNCHEOF`, a branch to
`EVNTEOF`, so the code runs exactly once. When `SOURCE_RECORD_LIMIT` is
set (`READLIM`, `P1ESRLIM`) the variant `MDLESRL` is used instead:

```asm
MDLESRL  BC    0,0(,0)             NOP (4 BYTES OF FILLER)
MDLESRLA LAY   R14,0(,R2)          -> PL8 limit in literal pool (CSV1OFF)
         clc   GPRECCNT,0(R14)     READ LIMIT REACHED ???
         BNL   EVNTEOF_indirect    YES - TREAT AS END-OF-FILE
mdlesrl_exit B  EVNTPREV           READ  NEXT EVENT RECORD
```

## 6.7 PASS1 (`GVBMR96`)

`PASS1` walks the in-memory logic table row by row (`P1LOOP`) with
R3 = current code-buffer position, R4 = end of generated code, R2 = current
NV code-segment address, R5/R6 = literal pool current/next. Its documented
responsibilities are:

1. convert TRUE/FALSE row numbers to row addresses;
2. save record-buffer addresses;
3. copy previous temporary literal pools into the code buffer;
4. copy model-code skeletons into the code buffer;
5. copy literals/constants into the temporary literal pool;
6. insert constant offsets and lengths into the skeletons.

Per-row processing (`P1INSERT` onward) is:

```text
P1TRUE / P1FALSE     resolve LTTRUE/LTFALSE row numbers -> addresses,
                     set LTOMITTB/LTOMITFB/LTOMITGO when a branch falls
                     through to the next row and can be omitted
P1LKUPBF             allocate/locate the lookup buffer for LU/LK/xxL rows
                     (p1look_up_alloc; second buffer for "LL" functions)
P1TRACE              insert BAS R10,MR95TRAC if TRACE applies to this row
P1LKUPRE             insert LKUPPREF (and lkup_number2) if LTLKUPRE is set,
                     insert `loadprev` if an operand comes from the
                     previous record ("P")
copy model           EX R15,MVCMODEL from FCMODELA for FCCODELN bytes
P1RELOLP             for each (CS*, offset) pair branch through P1RELOTB
                     to the P1xxxx routine that computes and stores it
P1NEXT               advance R3/R4, R8 -> next row
```

Rows whose `FCCODELN` is zero (`DIM`, `HD`, `RE`, pure control rows) skip
code generation entirely, and a row flagged `LTOMITGO` (a `GOTO` to the
physically next row) generates nothing.

### 6.7.1 Lookup prefix

When a row's operand comes from a lookup record (`LTLKUPRE`), PASS1
inserts the `LKUPPREF` code sequence in front of the skeleton. `LKUPPREF`
is generated by the `lkupcode` macro (`MAC/LKUPCODE.mac`) and is a fixed
`lkuppref_length` bytes:

```asm
         llgt R5,0(,R2)                  LOAD LOOK-UP BUFFER ADDRESS
         lg   R5,LBLSTFND-LKUPBUFR(,R5)  LOAD LOOK-UP RESULT ADDRESS
```

PASS1 stores the lookup buffer's address in the literal pool (`SAVELBA`),
keeps the resulting offset in `LTLUBOFF` (flag `ltlkup_offset`) and patches
it into the `llgt` displacement; at run time R5 then points at the record
the most recent `LU` for that buffer found (`LBLSTFND`, which the search
routines set to the "not found" default record when there is no match).
Functions with sub-function `LL` (two lookup operands, except `MULL`) get
a second prefix `lkup_number2`, identical except that it loads R1 from the
second buffer. PASS2
knows to skip `lkuppref_length` (twice for `LL`) and `L'loadprev` when
locating the skeleton inside the segment.

### 6.7.2 Branch omission

`P1TRUE`/`P1FALSE` compare the resolved TRUE and FALSE targets with the
physically next row. If a branch simply falls through it is *omitted* -
`LTOMITTB`/`LTOMITFB` is set and PASS2 leaves the optional
(`CSTRUEO`/`CSFALSEO`) branch instruction unpatched, i.e. as the
`jlu *+l'*` NOP the skeleton was written with. Mandatory branches
(`CSTRUEM`/`CSFALSEM`) are always patched. A `GOTO` whose target is the
next row sets `LTOMITGO` and generates no code at all.

## 6.8 PASS2 (`GVBMR95`)

After `GVBMR96` returns, `GVBMR95` runs `PASS2` on the main task before
any thread is attached:

```asm
PASS2    stg   R14,SAVF4SAG64RS14
         llgt  R8,LTBEGIN
P2LOOP   llgt  R3,LTCODSEG        code segment address for this row
         ... position R3 past trace BAS / LKUPPREF / loadprev ...
P2FUNCTB llgt  R6,LTFUNTBL
         TM    LTFLAGS,LTOMITGO
         JO    P2NEXT
         ltgf  R14,FCRELOCA
P2RELOLP CLI   0(R14),X'FF'
         JE    P2NEXT
         llgc  R1,1(,R14)         offset -> R1 = code base + offset
         llgc  R0,0(,R14)         CS* code
         select cij,r0,eq
           when 12  (CSTRUEO)  if not LTOMITTB: BRAS R9,TRUEOFF ; ST R0,0(,R1)
           when 13  (CSTRUEM)  BRAS R9,TRUEOFF  ; ST R0,0(,R1)
           when 14  (CSFALSEO) if not LTOMITFB: BRAS R9,FALSEOFF; ST R0,0(,R1)
           when 15  (CSFALSEM) BRAS R9,FALSEOFF ; ST R0,0(,R1)
           when 29  (CSTTLOFF) title offset = EXSRTKEY + LTSORTLN + LTFLDPOS
         endsel
         aghi  R14,2
         J     P2RELOLP
P2NEXT   agh   R8,0(,R8)          next row (LTROWLEN)
         cgf   R8,LTENROWA
         JNH   P2LOOP
```

`TRUEOFF`/`FALSEOFF` (`FALSECOM`) compute the displacement from the branch
instruction to `LTCODSEG` of the `LTTRUE`/`LTFALSE` row -
`(target - (R1 - 2)) >> 1`, i.e. halfwords relative to the start of the
`jlu` - so any target in the segment is reachable; a target that is an
`NV` or `ES` row is special-cased (`FALSENV`) because those rows' code is
the view prologue / set epilogue rather than a row skeleton. Before the relocation loop PASS2 also:

* `P2FUNNV` - disables views not listed in `RUNVIEWS`, patches `NVLAYR8`;
* `P2FUNES` - patches `MDLES`/`mdlesrl` with `BRNCHEOF` when the set has
  no event file.

If `DUMP_LT_AND_GENERATED_CODE=Y` the finished logic table and code
buffers are then `SNAP`ped to `EXTRDUMP`/`REFRDUMP`.

## 6.9 Run-time entry

At run time a thread enters its generated code from `EVNTCODE`
(see [03-gvbmr95-extract-engine.md](03-gvbmr95-extract-engine.md)):

```asm
evntnorec llgt  R7,GPEXTRA        extract record buffer
          llgt  r8,THRDES         ES row
          LLGT  R2,THRDLITP       literal pool
          agfi  r2,f512k          ... biased by 512K
          llgt  R15,LTESCODE      first NV code segment
          BR    R15
```

and the generated code runs NV prologue -> row code -> ... -> `MDLES`,
which branches back to `EVNTPREV` for the next record. All branches inside
the segment are relative, so the whole code buffer is position-independent
and can live anywhere below the bar.

## 6.10 Trace

With `TRACE` active (`EXECTRAC='Y'`, optionally narrowed per view by
`VIEWTRAC`/`LTPARMTB`), each traced row is preceded by `BAS R10,MR95TRAC`
which reaches `TRACSUBR` in `GVBMR95` via the `TRACE` stub. `TRACSUBR`
first switches the thread back to TCB mode if it is running on a zIIP SRB
(`TCB_switch`), then finds the current view's `NV` row through `LPVADDR`
in the literal pool header (kept current by the NV prologue - "this
possible call to trace MUST follow the lpvaddr update"). It honours the
`TRACE` parameter table (`PARMTBL`: per-view `VIEWTRAC`, `PARMFROM`/
`PARMTHRU` event-record range, and the `LTEVENTD` hex-dump option) and
writes formatted lines - row number, function code, operand values
(decimal-floating-point values are edited with `dfpmask158`/`dfpmask203`)
and optionally a hex dump of the event record - to `EXTRTRAC`
(`RECFM=FBA, LRECL=161`).
