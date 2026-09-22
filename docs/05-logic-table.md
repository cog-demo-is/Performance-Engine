# 5. The logic table

The logic table is the "program" that the Performance Engine executes. It is
produced by the GenevaERS workbench (or the `MR91` compiler, not in this
repository) from view definitions, written to a sequential file, read by
`GVBMR96` from `EXTRLTBL`, converted into an in-memory table of `LOGICTBL`
rows, and finally compiled into machine code (see
[06-generated-code.md](06-generated-code.md)). This document covers the
external row formats, the in-memory row, the function codes and the way rows
are chained.

```text
   EXTRLTBL (VB records, one per row)
        |
        |  GVBLT??A DSECTs  (MAC/GVBLTF1A.mac, GVBLTNVA.mac, ...)
        v
   LTBLLOAD (GVBMR96)
        |
        |  LOGICTBL DSECT  (MAC/GVBMR95L.mac): fixed prefix + redefinition
        v
   in-memory logic table  LTBEGIN ... LTEND
        |
        |  PASS1 / PASS2
        v
   generated code (one segment per row, LTCODSEG)
```

## 5.1 Input record formats (`MAC/GVBLT*.mac`)

Every input record starts with the same 44-byte header. Using `LTF1_REC`
as the example:

```asm
LTF1_REC             DSECT
LTF1_FILE_ID          DS FL04     event file id (LF id) of the ES set
LTF1_ROW_NBR          DS FL04     logic table row number
LTF1_VIEW_ID          DS FL04     view number
LTF1_FUNCTION_CODE    DS 0CL04
LTF1_MAIN_FUNCTION    DS CL02     e.g. 'CF'
LTF1_SUB_FUNCTION     DS CL02     e.g. 'EC'
LTF1_SOURCE_SEQ_NBR   DS HL02
LTF1_SUFFIX_SEQ_NBR   DS HL02
LTF1_GOTO_ROW1        DS FL04     TRUE  branch row number
LTF1_GOTO_ROW2        DS FL04     FALSE branch row number
LTF1_RECORD_TYPE      DS FL04     which layout follows
```

The record type selects one of these layouts:

| Macro | DSECT | Used by | Payload |
|-------|-------|---------|---------|
| `GVBLTGEN` | `LTGN_REC` | `GEN` | ASCII flag, extract-phase flag, generation date/time |
| `GVBLTHDA` | `LTHD_REC` | `HD` | date and time stamp of the logic table |
| `GVBLTNVA` | `LTNV_REC` | `NV` | view type, source LR id, little-endian flag, sort-key / sort-title / DT-area lengths, CT column count, max extract records |
| `GVBLTREA` | `LTRE_REC` | `RE*` | read-exit id |
| `GVBLTF0A` | `LTF0_REC` (= `LTES_REC`, `LTEN_REC`, `LTNMF1_REC`) | `ES`, `EN`, `ET`, `GO`, `NO`, `JOIN` … | header only |
| `GVBLTF1A` | `LTF1_REC` | one-field rows (`DTE`, `SKE`, `LKE`, `CFEC`, `SETE`, …) | column id, compare type, a 260-byte **value area** (length + 256-byte value) and an 84-byte **field area** (file/LR/field ids, format, sign, start position, length, ordinal position, decimals, rounding, content id, justify id, 48-byte mask) |
| `GVBLTF2A` | `LTF2_REC` | two-field rows (`CFEE`, `CFEL`, `SFEL`, …) | two value areas and two field areas |
| `GVBLTCCA` | `LTCC_REC` | `CFCC` | two constant values, compare type, field content id |
| `GVBLTV1A` | `LTV1_REC` | accumulator rows with one operand (`ADDE`, `SETC`, `DTA`, …) | same as F1 plus a 48-byte accumulator name |
| `GVBLTV2A` | `LTV2_REC` | accumulator rows with two operands (`FNEC`, `SFEC` …) | same as F2 plus accumulator name |
| `GVBLTVNA` | `LTVN_REC` | `DIM` | variable (table) name, value length, value |
| `GVBLTVVA` | `LTVV_REC` | variable value / compare | table name, value, compare type |
| `GVBLTWRA` | `LTWR_REC` | `WR*` | destination type, output file id, write-exit id, summarised record count, 256-byte exit parameter string |

`MAC/GVBLTV3.mac` maps the older (version 3) fixed-format logic table
(`LTBLV3`), which `GVBMR96` can still read; the fields (`V3FUNC`,
`V3FLDPOS`, `V3TRUE`, `V3FALSE`, …) are converted to the same in-memory
`LOGICTBL` rows.

The `field area` is the important part: it tells the generator where an
operand lives (`START_POSITION`/`FIELD_LENGTH` relative to the event,
lookup or previous record), its data type (`FIELD_FORMAT` — one of the
`FC_*` format codes below), whether it is signed, how many decimals it has,
and the `CONTENT_ID` (date/time content code used by the `FN*` date
functions and formatting).

## 5.2 The in-memory row (`LOGICTBL`, `MAC/GVBMR95L.mac`)

`LTBLLOAD` converts every input record into a variable-length `LOGICTBL`
row. All rows share a fixed prefix; the remainder (`LTREDEFN`) is an `ORG`
redefinition chosen by function code.

```asm
LOGICTBL DSECT
LTROWLEN DS    HL02     row length (rounded to fullword)
LTFLAG1  DS    XL01
  LTESCLON      X'80'   cloned ES set
  LTLKUPRE      X'40'   look-up prefix needed before the skeleton
  ltlkup_alloc  X'20'   look-up buffer allocated, address in LTLBADDR
  ltlkup_offset X'10'   LTLUBOFF is valid
  LTOMITTB      X'08'   TRUE  branch omitted from generated code
  LTOMITFB      X'04'   FALSE branch omitted from generated code
  LTOMITSR      X'02'   shift & round omitted
  LTOMITGO      X'01'   GOTO branch omitted
LTFLAG2  DS    XL01
  LTRTOKEN      X'80'   event "file" is a token
  LTNOTOPT      X'40'   look-up not optimisable
  LTOMITLD      X'20'   first load instruction omitted
  LTLOCAL       X'10'   local (view) variable
  LTNOMODE      X'08'   exit may switch AMODE
  LTPIPEOF      X'04'   pipe writer EOF signalled
  LTEXTSUM      X'02'   extract summarisation exists
  LTEVENTD      X'01'   include event record dump in trace
LTROWNO  DS    FL04     row number
LTVIEW#  DS    FL04     view number
LTFUNC   DS    0CL04
LTMAJFUN DS    CL02     major function ('CF', 'DT', 'LK', ...)
LTSUBFUN DS    CL02     sub-function   ('EC', 'E ', 'C ', ...)
LTTRUE   DS    AL04     -> row taken when the condition is TRUE
LTFALSE  DS    AL04     -> row taken when FALSE
LTFUNTBL DS    AL04     -> FUNCTBL entry selected for this row
LTLOGNV  DS    AL04     -> logical NV row (view) owning this row
LTVIEWNV DS    AL04     -> NV row of the *view* (differs for CALL VIEW)
LTCODSEG DS    AL04     -> generated code segment for this row
LTACADDR DS    0AL04    accumulator address           (overlaid)
LTLBADDR DS    0AL04    look-up buffer address        (overlaid)
LTLUBOFF DS    0AL04    look-up buffer offset in pool (overlaid)
LTWREXTO DS    AL04     extract file control offset   (overlaid)
LTGENLEN DS    HL02     estimated size of generated code + literals
LTseqno  DS    H        sequence / column number (trace)
LTREDEFN DS    0C       ---- redefinition area ----
```

`LTTRUE`/`LTFALSE` hold **row numbers** after `LTBLLOAD` and **row
addresses** after PASS1; PASS2 turns them into branch displacements.

### Redefinitions

| Redefinition | Length symbol | Rows | Key fields |
|--------------|---------------|------|------------|
| (none) | `LTF0_LEN`, `LTGO_LEN`, `LTEN_LEN` | `GO`, `EN`, `NO`, `JOIN` | — |
| Variable name | `LTVN_LEN` | `DIM` | `LTVNFMT`, `LTVNCON`, `LTVNNDEC`, `LTVNSIGN`, `LTVNNAME` (48), `LTVNNEXT` (chain of variables), `LTVNLITP` (offset of the variable in the literal pool), `LTVNCOL#`, `LTVNLEN`/`LTVNVAL` |
| Variable value | `LTVV_LEN` | variable compare/assign | `LTVVFMT`, `LTVVLEN`, `LTVVROPR`, `LTVVVAL` |
| Format 1 | `LTF1_LEN` | one-operand rows | `LTCOLNO`, `LTCOLID`, `LTFLDDDN`/`LTFLDFIL`, `LTFLDLR`, `LTFLDPTH` (join path), `LTFLDID`, `LTFLDPOS`, `LTFLDSEQ`, `LTFLDLEN`, `LTFLDFMT`, `LTFLDCON`, `LTNDEC`, `LTRNDFAC`, `LTSIGN`, `LTSRTSEQ`, `LTRELOPR` (relational operator), `LTSTRNO`/`LTSTRLVL`, `LTAREAID`, `LTLKUPOS`/`LTDATPOS`, `LTJUSOFF`, `LTLUSTEP`, `LTV1LEN`, `LTV2LEN`, `LTVALUES` (constants follow) |
| Format 2 | `LTF2_LEN` | two-operand rows and column targets | Format 1 fields plus the target column: `LTCOLFIL`, `LTCOLLR`, `LTCOLPTH`, `LTCOLFLD`, `LTCOLPOS`, `LTCOLSEQ`, `LTCOLLEN`, `LTCOLFMT`, `LTCOLCON`, `LTCOLDEC`, `LTCOLRND`, `LTCOLSGN`, `LTCOLJUS`, `LTCOLGAP`, `LTAC2ADR` (second accumulator), occurrence fields `LTCOLOCC`/`LTOCCPOS`/`LTOCCLEN`/`LTOCCFMT`/`LTSEGLEN`, `LTMSKLEN`/`LTCOLMSK` |
| Value 1 / Value 2 | `LTV1_LEN`, `LTV2_LEN` | constants | `LTV1MLEN`, `LTV1VLEN`, `LTV1MSK`, `LTV1VAL` (and the `LTV2*` equivalents) |
| Header | `LTHD_LEN` | `HD` | `LTRUNDT`/`LTRUNNO`, `LTPROCDT`, `LTPROCTM`, `LTFINPDT`, `LTMAXFIL` (standard extract file count) |
| New view | `LTNV_LEN` | `NV` | `LTVIEWTP`, `LTSTATUS`, `LTUSERID`, `LTLRID`, `LTSORTLN`, `LTTITLLN`, `LTDATALN`, `LTMAXCOL`, `LTMAXOCC`, `LTSUMCNT`, `LT1000A` (-> VDP 1000 record), `LTNVVNAM`, `LTNVTOKN`, `LTNEXTNV`, `LTFRSTLU`, `LTFRSTWR`, `LTVIEWRE`, `LTVIEWES`, `LTPARENT`, `LTPARMTB` (trace parms), `LTSUBOPT`, `LTNVRELO`, `LTENDIAN`, `LTMINCOL` |
| Read event | `LTRE_LEN` | `RE*` | `LTFILTYP` (access method), `LTFILEDD`/`LTFILEID`, `LTREPFcnt`, `LTREEXIT`, `LTRENAME`, `LTREADDR`/`LTREENTP`/`LTREWORK`, `LTREPARM` (32), `LTNEXTRE`, `LTREES` (-> owning ES), `LTCLONRE`, `LTVDP200` (-> VDP 0200 record, 8 bytes), `LTTOKNLR`, `WRTKNCNT`, `LTFILCNT` (records read, 8 bytes), `LTHDROPT`, `LTVERNO`, `LTCTLCNT`, `LTEDITPR`, `LTOBJID`, `LTACCMTH`, `LTTRACNT`, `LTVOLSER`, `LTDDNAME`, `LTDSNAME` |
| Look-up | `LTLU_LEN` | `LU*` | `LTLUFILE`/`LTLUFID`, `LTLULRID`, `LTLUPATH`, `ltlu_next_exit`, `LTLUEXIT`, `LTLUNAME`, `LTLUptyp` (exit language 1–5), `LTLUADDR`/`LTLUENTP`/`LTLUWORK`, `LTLUPARM`, `LTLUWPATH`, `LTLUopt`/`flg2`/`flg3`/`flg4`, `LTLUNEXT` |
| Write | `LTWR_LEN` | `WR*` | `LTWREXT#` (extract file number), `LTWRFILE`/`LTWRFID`, `LTWR200A`, `LTWREXIT`, `LTWRNAME`, `LTWRADDR`/`LTWRENTP`, `LTWRPARM`, `LTWRLUBO`, `LTWREXTA` (-> `EXTFILE`), `LTWRDEST`, `LTWRSUMC`, `LTWRLMT`, `LTWRNEXT`, `LTWRVIEW`, `ltwrtkrc`; `LTWRPGM_TYPE_*` = LE COBOL (1), COBOL2 (2), C (3), C++ (4), Java (5), ASM (6) |
| End of set | `LTES_LEN` | `ES` | `LTESSET#`, `LTTHRDWK` (-> `THRDAREA`), `LTESVNAM`, `LTPIPELS`, `LTNEXTES` (dispatch chain), `LTFRSTRE`, `LTFRSTNV`, `LTTOKNNV`, `LTSAMES#`, `LTCLONES`, `LTESINIT`, `LTESCODE` (-> first generated instruction of the set), `LTESLPSZ`/`LTESLPAD` (literal pool size / address), `ltesrtkq` (RETK/RETX queue), `ltesretn`/`ltespr11` (ET return), `LTLBANCH` (look-up buffer chain anchor), `ltesflg1` (`ltlbanco` = chain anchored via literal-pool offset), `ltes_time` (accumulated CPU time) |
| Compare constant | `LTCC_LEN` | `CFCC` | `LTCCLEN1`, `LTCCLEN2`, `LTCCVAL1`, `LTCCVAL2` |
| Error / overflow fill | `LTP1_LEN`, `LTP2_LEN` | view fill values | `LTERRLEN`/`LTERRFIL`, `LTOVRLEN`/`LTOVRFIL` |

The `RE`, `LU` and `WR` redefinitions deliberately place their
"next exit" pointer at the same offset (`ltlu_next_exit` is preceded by a
2-byte pad for this reason) so that exit-loading code can walk all three
kinds of exit row with one loop.

## 5.3 Function codes

A function code is four characters: a two-letter **major function** and a
two-letter **sub-function**. For operand-bearing functions the sub-function
letters name the operand sources:

| Letter | Source (`FC_*` operand type in `MAC/GVBMR95C.mac`) |
|--------|-----------------------------------------------------|
| `C` | constant in the literal pool (`FC_CON`) |
| `E` | event record (`FC_EVNT`) |
| `L` | look-up record (`FC_LKUP`; a second `L` selects `FC_LKUP2`) |
| `P` | previous event record (`FC_PREV`) |
| `X` | prior column of the extract record being built (`FC_PRIOR`) |
| `A` | numeric accumulator (`FC_ACUM`) |
| `T` / `K` | calculated-column area (`FC_CTAR`) / sort key (`FC_SKEY`) |

The complete list of major functions, as ordered in `GVBmaj_t`
(`ASM/GVBMR95.asm`) and recognised by `LTBLLOAD`:

| Major | Meaning | Notes |
|-------|---------|-------|
| `GEN` | generation record | first record; date/time, extract flag |
| `HD` | header | date/time stamp, run number |
| `NV` | new view | starts a view within an ES set; owns the DT/CT/sort-key areas |
| `RE` | read event file | `RENX` (no exit), `REEX` (read exit), `RETK` (token), `RETX` (token + exit) |
| `ES` | end of event set | last row of a set; generated code branches back to `EVNTPREV` in `GVBMR95` |
| `ET` | end of token set | returns to the token writer |
| `EN` | end of logic table | |
| `LK*` | build look-up key | `LKC`, `LKE`, `LKL`, `LKP`, `LKS` (symbol), `LKLR` (LR id), `LKDC`/`LKDE` (effective date from constant/event), `LKD` |
| `JOIN` | join step | LR-id operand |
| `LU` | perform look-up | `LUSM` (search memory-resident table), `LUEX` (look-up exit) |
| `KS` | key save | `KSLK` saves the look-up key |
| `CF*` | compare field | `CFEC`, `CFLC`, `CFPC`, `CFXC` (field vs constant), `CFEA`/`CFLA`/`CFPA`/`CFXA`/`CFCA` (vs accumulator), `CFCC` (constant range), plus two-field `CFEE`, `CFEL`, … resolved through the `cf_array` |
| `CN`, `CS`, `CX` | class tests | is-numeric (`CNE`), is-spaces (`CSE`), is-low-values (`CXE`) |
| `SF*` | substring compare | `SFEC`, `SFLC`, `SFPC`, `SFXC`, `SFC`, `SF` — "needle in haystack" `CLC` loop (`mdlsfxc`) |
| `FN*` | date functions | `FNEC`, `FNLC`, `FNPC`, `FNXC`, `FNCC`, `FNC`, `FN` — days/months/years between (`fnxcsub_indirect`, `fnccsub_indirect`, using `GVBDAYS`) |
| `DIM` | declare variable | creates a named accumulator/string variable |
| `SET`, `ADD`, `SUB`, `MUL`, `DIV` | accumulator arithmetic | `SETA`/`SETC`/`SETE`/`SETL`/`SETP`, `ADDA`/`ADDC`/`ADD*`, … ; the operand format pair selects the entry from a format array |
| `DT*` | data column | move a value into the extract record's DT area (`DTA`, `DTC`, `DTE`; the `DTE` entries of `dt_array` are reused for look-up/previous/prior sources, the operand type being set from the sub-function letter) |
| `CT*` | calculated column | move/accumulate into the CT area (`CTA`, `CTC`, `CT*`) |
| `SK*` | sort key | `SKC`, `SKE`, `SKL`, `SKP`, `SKA` |
| `WR*` | write | `WRXT` (extract record), `WRDT` (DT area), `WRIN` (event record as-is), `WRSU` (summarised), `WREX` (write exit), `WRTK`/`WRTX` (token) |
| `GO`, `NO` | `GOTO`, `NOOP` | unconditional branch / no operation |

The `major_func_table` entry (`MAC/MAJRFTAB.mac`) records for each code a
prefix length, the two operand types, a `major_index` (the `case` value
used by `LTBLLOAD`'s `select`) and a pointer that is either a single
`FUNCTBL` entry, a 13-entry *vector* indexed by the one variable operand
format, or the 4×13×13 *array* indexed by target format, source format and
the two signs. The 13 format positions are:

```text
 0 default   1 FC_ALNUM  2 FC_ALPHA  3 FC_NUM   4 FC_PACK  5 FC_SORTP
 6 FC_BIN    7 FC_SORTB  8 FC_BCD    9 FC_MASK 10 FC_EDIT 11 FC_FLOAT
12 FC_GEN#
```

(`FC_ALPHA` and `FC_GEN#` columns are always zero; `FC_FLOAT` is a 16-byte
DFP extended value — the internal accumulator format since 2014.)

The `FC_RTYP` byte of a `FUNCTBL` entry says which input-record layout
the row uses (`FC_RTYP01` header, `02` new view, `03/04/05` format 0/1/2,
`06` read event, `07` write, `08` compare constant, `09` variable name, `0A`
variable value, `0B` variable calculation, `0C/0D` variable function
format 1/2).

## 5.4 Chains through the table

The in-memory table is a linear sequence of rows (`LTBEGIN` to `LTEND`, each
row `LTROWLEN` long), but the engine navigates it mostly through embedded
pointers:

```text
 THRDAREA.LTNXDISK / LTNXTAPE / LTNXOTHR      (dispatch queues, GVBMR95)
        |
        v
   +---------+   LTNEXTES    +---------+   LTNEXTES
   |  ES #1  | ------------> |  ES #2  | ------------> ...
   +---------+               +---------+
     | LTFRSTRE                 | LTTHRDWK -> THRDAREA
     v                          | LTESCODE -> generated code
   +---------+   LTNEXTRE       | LTLBANCH -> look-up buffers
   |  RE     | ------------> ...
   +---------+
     | LTREES (back to ES)
   ES.LTFRSTNV
     |
     v
   +---------+   LTNEXTNV    +---------+
   |  NV #1  | ------------> |  NV #2  | -> ...
   +---------+               +---------+
     | LTFRSTLU -> first LU row (-> LTLUNEXT -> ...)
     | LTFRSTWR -> first WR row (-> LTWRNEXT -> ...)
     | LTVIEWRE / LTVIEWES -> owning RE / ES
     | LTPARENT -> parent NV when this view is CALLed
     | LTPARMTB -> trace parameters
```

Within a view, control flow is expressed only through `LTTRUE`/`LTFALSE`:
a compare (`CF*`) row branches to one or the other; unconditional rows use
`LTTRUE` as "next"; `GO` uses it as its target. PASS1 may set `LTOMITTB`,
`LTOMITFB` or `LTOMITGO` when a branch target is simply the next row, so
that no branch instruction is emitted for it.

## 5.5 Cloned sets

When `CLONLTBL` clones an ES set for a partitioned event file, the cloned
rows are flagged `LTESCLON`; `LTCLONRE`/`LTCLONES` link an original to its
copy and `LTSAMES#` groups all copies of the same logical set. Each clone
has its own `LTTHRDWK`, `LTESCODE` and literal pool — from the sub-task's
point of view a clone is just another independent event set.

## 5.6 Where to look in the source

| Topic | Location |
|-------|----------|
| In-memory row DSECT | `MAC/GVBMR95L.mac` |
| Input record DSECTs | `MAC/GVBLTF0A.mac` … `MAC/GVBLTWRA.mac`, `MAC/GVBLTGEN.mac`, `MAC/GVBLTV3.mac` |
| Function code constants | `ASM/GVBMR96.asm` (`HD`, `RE_NX`, …, `EN`) |
| Major function table / operand types | `ASM/GVBMR95.asm` `GVBmaj_t`, `MAC/MAJRFTAB.mac`, `MAC/GVBMR95C.mac` |
| Row conversion | `ASM/GVBMR96.asm` `LTBLLOAD` (`select ... case nn` on `major_index`) |
