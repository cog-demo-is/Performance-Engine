# 4. GVBMR96 — initialization and code generation

`ASM/GVBMR96.asm` (~19k lines) is link-edited into the same load module as
`GVBMR95` (see `LINKPARM/GVBMR95.ftl`) and is called exactly once, from the
main task, before any sub-task is attached. Its job is to turn the external
inputs (execution parameters, environment variables, the VDP, the logic
table and the reference-data files) into the in-memory structures that the
generated code and the sub-tasks will use, and then to run **PASS1** of the
code generator. It is `RMODE ANY` / `AMODE 31`, but many of its routines
switch to AMODE 64 (`sam64`/`sam31`) to touch the 64-bit lookup and
literal-pool storage.

## 4.1 Call sequence

The mainline of `GVBMR96` is a straight list of subroutine calls. Each
routine is a self-contained phase; the order matters because later phases
consume pointers set by earlier ones.

```asm
jas   R14,ENVVLOAD      environment variables (EXTRENVV)
jas   R14,Parmload      execution parameters  (EXTRPARM)
jas   R14,EDITParm      validate / normalise parameters
BRAS  R9,PRNTRPT        open control report, print header
Larl  R15,VDPLOAD  ...  load the VDP (MR95VDP)
Larl  R15,LTBLLOAD ...  load and convert the logic table (EXTRLTBL)
larl  R15,INIT_IO  ...  initialise I/O driver modules
larl  R15,CLONLTBL ...  clone "ES" sets for multi-partition event files
larl  R15,MEMALLOC ...  allocate work areas
larl  R15,LOADLKUP ...  load reference (lookup) tables (EXTRREH / EXTRRED)
larl  R15,THRDBLD  ...  build one THRDAREA per event-file set
larl  R15,ALLOCLIT ...  allocate literal-pool work areas
larl  R15,PASS1    ...  PASS1 of code generation
larl  R15,OPENEXTF ...  open extract files
```

```text
   EXTRENVV   EXTRPARM   MR95VDP    EXTRLTBL     EXTRREH/EXTRRED
       |          |          |          |               |
       v          v          v          v               |
   ENVVLOAD -> Parmload -> VDPLOAD -> LTBLLOAD -> INIT_IO -> CLONLTBL
                 |                                              |
              EDITParm                                          v
                                                            MEMALLOC
                                                                |
                                                                v
                                                            LOADLKUP <----+
                                                                |
                                                                v
              THRDBLD -> ALLOCLIT -> PASS1 -> OPENEXTF -> return to GVBMR95
                                                          (which runs PASS2)
```

## 4.2 ENVVLOAD — environment variables

`ENVVLOAD` reads the optional `EXTRENVV` file (DCB `ENVVDCB`) into an
environment-variable table. Each record is a `NAME=VALUE` pair; the table is
used later by `GVBUR33` to substitute `&NAME.` style symbols in DSNs,
parameters and DB2 SQL text (see [11-utilities.md](11-utilities.md)). The
table address is stored in `THRDAREA` so it is visible to every thread.

## 4.3 Parmload / EDITParm — execution parameters

`Parmload` reads `EXTRPARM` (DCB `PARMDCB`) and `EXTRTPRM` (DCB `TPRMDCB`,
trace parameters) record by record, tokenising `KEYWORD=VALUE` pairs and
matching each keyword against `PARMKWRD_table` (35-byte keyword, 2-byte
index). The recognised keywords and their default values (from `STDPARMS`)
are listed in [13-parameters-ddnames-messages.md](13-parameters-ddnames-messages.md).
The results are stored in the `EXECDATA` control block (`MAC/EXECDATA.mac`)
whose address is `EXECDADR` in `THRDAREA`. The layout of `STDPARMS` is
asserted to match `EXECDATA`:

```asm
STDPARML EQU   *-STDPARMS
         ASSERT stdparml,eq,execdlen     these two MUST match
```

`EDITParm` then validates the values — numeric fields are packed, `Y/N`
switches are checked, thread limits are capped, and inconsistent
combinations produce messages via `GVBMSG`.

The `TRACE` parameter has its own keyword table (`TraceParmtable`: `VIEW`,
`FROMREC`, `THRUREC`, `FROMLTROW`, ...) and populates a per-view trace
parameter table pointed to by `LTPARMTB` in each `NV` row. A trace request
also enlarges the generated code (see §4.10).

## 4.4 VDPLOAD — the View Definition Parameters

`VDPLOAD` reads the `MR95VDP` file (DCB `VDPDCB`). Every VDP record shares
the header mapped by `MAC/VDPHEADR.mac`:

```asm
vdp_header      DSECT
vdp_REC_LEN       DS H        RDW length
vdp_RDW_FLAGS     DS XL02
vdp_VIEW_no       DS F
vdp_INPUT_FILE_ID DS F
vdp_COLUMN_ID     DS F
vdp_RECORD_TYPE   DS H        1, 2, 50, 100, 200, 210, 300, ...
vdp_SEQUENCE_NBR  DS H
vdp_RECORD_ID     DS F
```

The record types and the macro that maps each one:

| Type | Macro | Meaning |
|------|-------|---------|
| 0001 | `GVB0001A` | Generation record: run number/date, endian/ASCII indicators, counts of every other record type |
| 0002 | `GVB0002A` | Format-phase views list |
| 0050 | `GVB0050A` | Control record |
| 0100 | `GVB0100A` | Server record |
| 0200 | `GVB0200A` | Physical file (DSN, DDNAMEs, DBMS subsystem/table/SQL, allocation attributes, access-method id) |
| 0210 | `GVB0210A` | Program (exit) file record |
| 0300 | `GVB0300A` | Logical record (LR) |
| 0400 | `GVB0400A` | LR field |
| 0500 | `GVB0500A` | LR index (key) |
| 0600/0601 | `GVB0600A`/`GVB0601A` | Join (lookup path) step |
| 0650 | `GVB0650A` | Join names |
| 0700 | `GVB0700A` | Call (exit) parameter |
| 0800/0801 | `GVB0800A`/`GVB0801A` | Extract file(s) |
| 1000 | `GVB1000A` | View: name, type, status, output media/destination, page/line sizes, as-of dates, fill values, exits |
| 1200/1210 | `GVB1200A`/`GVB1210A` | Summary record logic / calculation |
| 1300 | `GVB1300A` | Title lines |
| 1400 | `GVB1400A` | Footer lines |
| 1600 | `GVB1600A` | Summary output file |
| 2000 | `GVB2000A` | Column |
| 2200/2210 | `GVB2200A`/`GVB2210A` | Summary column logic / calculation |
| 2300 | `GVB2300A` | Sort key attributes |
| 3000 | `GVB3000A` | View source LR/file |
| 3200 | `GVB3200A` | LR-file logic |
| 4000 | `GVB4000A` | LR-file column |
| 4200 | `GVB4200A` | File column logic |

`VDPLOAD` copies the records into a table sized by `EXECVSIZ` (`VDP TABLE
SIZE (K)`, default 1000K) and maintains, per record type, a pointer to the
first record and a chain of "next record of the same type". The source
comment notes that the in-memory VDP elements also carry a `_NEXT_PTR`
field for traversing *all* elements, which is needed because the table may
live in non-contiguous segments. Later phases resolve `LT*FID`, `LT*LRID`
and `LT*PATH` ids in the logic table against the 0200/0300/0650 records
(`LTVDP200`, `LTWR200A`, `LT1000A` hold the resulting addresses).

The 0001 record's date/time is saved in `THRDAREA` (`VDP_DATE`) so that
`LTBLLOAD` can compare it with the logic table's `GEN` record (see §4.5).

## 4.5 LTBLLOAD — loading and converting the logic table

`LTBLLOAD` reads `EXTRLTBL` (DCB `LTBLdcb`). Each input record is one row
of the workbench logic table (formats in
[05-logic-table.md](05-logic-table.md)).

The first record is the `GEN` record (`LTGN_REC`, `MAC/GVBLTGEN.mac`). It
carries the generation date/time, per-type row counts and the
`LTGN_Extract` flag. `LTBLLOAD` uses it to:

* decide the phase — `LTGN_Extract = X'01'` sets `extract_phase='Y'`;
  otherwise this is a reference-phase run and the standard extract file
  count `MAXSTDF#` is forced to zero (only `REH`/`RED` outputs are written);
* verify, unless `VERIFY_CREATION_TIMESTAMP=N`, that `LTGN_DATE`/`LTGN_TIME`
  match the VDP 0001 record's date/time (`VDP_XLT_TIMESTAMP_ERR` otherwise);
* size the in-memory table from `LTGN_reccnt`.

For every subsequent row it:

1. Identifies the record type by function code and maps it with the matching
   `GVBLT??A` DSECT.
2. Looks up the 4-byte function code in the **major function table**
   `GVBmaj_t` (assembled in `GVBMR95`, exported via `ENTRY GVBmaj_t` /
   `GVBmaj_c`). The table is searched by prefix; the `array_ent` macro
   records a sequence number for each entry that `GVBMR96` uses in
   `select`/`when` dispatch, which is why the source warns that new
   function codes must be *appended* and longer codes must precede shorter
   ones with the same prefix.
3. Allocates the corresponding `LOGICTBL` row (`MAC/GVBMR95L.mac`) in the
   in-memory logic table (`LTBEGIN` … `LTEND`), copying field attributes
   (position, length, format, content, decimals, rounding, sign, mask) from
   the input record into the `LTF1`/`LTF2` redefinitions, and constants into
   the `LTVALUES`/`LTV1VAL`/`LTV2VAL` areas.
4. Resolves the `FUNCTBL` entry (`LTFUNTBL`). Where the function code
   implies operand types (e.g. `ADDE` → second operand is the event record)
   `GVBMR96` sets them from the letter in the code, as the `GVBmaj_t`
   comment explains. For arithmetic/compare functions with variable operand
   types the entry address is selected from a **13×13 array** indexed by
   target format (row) and source format (column), replicated four times
   for the unsigned/signed combinations (see
   [06-generated-code.md](06-generated-code.md#62-the-major-function-table)).
   `LTBLLOAD` also chooses the `FCLTSUB` load subroutine (`FCSUBCOM`,
   `FCSUBDEC`, `FCSUBLE`, `FCSUB_LONG` for fields > 256 bytes, ...) that
   handles length/format compatibility between the two operands.
5. Tracks per-view minimum/maximum `CT` column numbers (`LTMINCOL`,
   `LTMAXCOL`) and counts of each row type.
6. **Estimates the generated code size** and the **literal-pool size** for
   the row (`ltblgenl`): starting from `FCCODELN`, adding a branch
   instruction if tracing is enabled for this view, adding `rdtokenl` for
   token readers, adding one or two `lkuppref` prefixes when a lookup
   buffer address must be loaded first (`LTLKUPRE`), and adding the length
   of any constant that must be copied into the literal pool (after
   `chkcons` checks whether an identical constant is already pooled). The
   results go to `LTGENLEN`, and are accumulated into `codesize` and
   `espoolsz`.
7. Rounds `LTROWLEN` up to a fullword and chains the row.

Row numbers in `LTTRUE`/`LTFALSE` (the `GOTO_ROW1/2` fields of the input
record) are left as numbers at this point; PASS1 converts them to addresses.

## 4.6 INIT_IO — I/O driver initialisation

`INIT_IO` gives each optional I/O module a chance to initialise. The DB2
modules are weak externals (`WXTRN GVBMRSQ`, `GVBMRSU`, `GVBMRDV`, and
`GVBMRDI` in `GVBMR96`), so their V-cons are tested for zero before calling.
The `EXEC_DB2HPU` flag is set when any `RE` row uses the `DB2HPU` access
method so that `GVBMRSU`/`GVBMRHPU` can be primed. Environment-variable
parsing errors detected here are reported through `GVBMSG`.

## 4.7 CLONLTBL — cloning `ES` sets for partitioned event files

An event "file" in the VDP may be a set of physical partitions (several
0200 records for the same logical file). To read partitions in parallel,
`CLONLTBL` walks the three `ES` dispatch chains (disk, tape, other — see
[08-threading-ziip-recovery.md](08-threading-ziip-recovery.md)) and, for
every additional partition, **copies the whole `RE … ES` group of logic
table rows** for that event set, flagging the copies with `LTESCLON` and
linking originals to copies through `LTCLONRE`/`LTCLONES`. Each copy gets
its own `LTVDP200` (partition record), its own thread work area and its own
generated code. The routine computes a *view relocation factor* so that the
`TRUE`/`FALSE` row numbers inside the copied rows still point into the copy.
Cloned `ES` sets that feed pipes increment the `EXTPIPEP` counter in the
pipe list so the pipe reader knows how many writers to expect.

## 4.8 MEMALLOC and THRDBLD — work areas

`MEMALLOC` obtains the large shared work areas: the extract-record buffer
area, the `DT`/`CT` (data / calculated-column) areas per view, the
accumulator area, the sort-key and title-key areas, and the lookup-buffer
(`LKUPBUFR`) chain anchored at `LTLBANCH` in the `ES` row.

`THRDBLD` then builds one `THRDAREA` per `ES` set (per event file or
partition), as its header comment lists:

```text
1  ALLOCATE THREAD  WORK  AREAS  FOR   EACH  EVENT FILE SET
2. ALLOCATE EVENT   FILE  DCB
3. ALLOCATE EXTRACT FILE  RECORD BUFFER
4. ALLOCATE THREAD  COMPLETION  "ECB"  LIST
```

Each thread area is a copy of the main `THRDAREA` template with its own
save areas, ECB, `GENPARM` block, DCB/ACB, extract buffers and counters
(layout in [12-control-blocks.md](12-control-blocks.md)). The areas are
chained through `THRDNEXT` from `THRDFRST`, `THRDCNT` is incremented and
the `ES` row's `LTTHRDWK` points back at the thread area. The ECB list built
here is what `GVBMR95` later hands to `EVENTS ENTRIES=(n)`.

## 4.9 LOADLKUP — loading reference data

`LOADLKUP` ("fill memory resident tables") reads the reference-phase output:
the header file `EXTRREH` (`REFRREH` under the `GVBMR95R` alias; one `REH`
record per lookup table giving key length, record count and data length)
and the corresponding reference data files, whose DDNAMEs come from the VDP
0650 record's `GREF` entries (`REFRxxx`, `GREFXXX` template). It obtains one
64-bit **reference pool** with `IARV64 REQUEST=GETSTOR` (`Refpoolb` /
`Refpoolc`), copies each table's records into it so that they are
contiguous and sorted by key, and fills in a lookup-buffer control block
(`LKUPBUFR`) per table. It then decides between the **binary-search**
routine for that key length (`SRCHADDT`, a 256-entry V-con table
assembled at the end of `GVBMR95.asm` from the `GVBSRCH` macro; the routines
themselves are in `GVBSRCHR`) and a **hash table** (when `HASH_TABLE_LU` / `HASH_PACK` /
`HASH_MULT` request it for the LF/LR pair), and stores the routine address
for PASS1 relocation code `CSSRCHR`. Details are in
[07-lookups-and-reference-data.md](07-lookups-and-reference-data.md).

## 4.10 ALLOCLIT — literal pools

Every `ES` set owns a literal pool: the generated code references
constants, addresses and counters as displacements from R2 (`THRDLITP` +
512K, see [03-gvbmr95-extract-engine.md](03-gvbmr95-extract-engine.md#38-the-record-loop)).
`ALLOCLIT` obtains the pool storage using the sizes accumulated by
`LTBLLOAD` (`espoolsz` per `ES` set, plus `LITPHDRL` for the `LITP_HDR`
header that starts every pool) and stores the pool address/size in
`LTESLPAD`/`LTESLPSZ`. Because the generator needs a *temporary* pool
while it is still deciding which constants are duplicates, PASS1 fills a
temporary pool and then copies it into the code buffer (item 3 of the
PASS1 list below).

## 4.11 PASS1 — first pass of code generation

`PASS1` walks every logic-table row and performs the six steps its header
documents:

```text
1.  CONVERT  "TRUE/FALSE" ROW NUMBERS  TO ROW ADDRESSES
2.  LOOK-UP   AND SAVE RECORD BUFFER   ADDRESSES
3.  COPY PREVIOUS  TEMPORARY LITERAL   POOL  INTO CODE  BUFFER
4.  COPY  MODEL CODE SEGMENT SKELETONS INTO  CODE BUFFER
5.  COPY  LITERALS/CONSTANTS TO TEMPORARY LITERAL POOL
6.  INSERT CONSTANT  OFFSETS  AND LENGTHS INTO  SKELETON
```

For each row (`P1FUNCCD` dispatches on the function code) the generic path
is:

```asm
llgt  R9,LTFUNTBL          function table entry
llgt  R14,FCMODELA         model code address
LH    R15,FCCODELN         model code length
BCTR  R15,0
EX    R15,MVCMODEL         copy skeleton into the code buffer at R3
llgt  R14,FCRELOCA         relocation table for this skeleton
LA    R4,1(R4,R15)         end-of-code position
P1RELOLP CLI 0(R14),X'FF'  end of relocation table?
         ...               branch through P1RELOTB on the CS* code
```

The relocation table is a list of `(code, offset)` byte pairs; `P1RELOTB`
is a 70-odd entry jump table (`CSSRCLN` → `P1SRCLN`, `CSTRUEO` →
`P1TRUEO`, `CSLBAOFF` → `P1LBAOFF`, ...) whose targets patch operand
lengths, base/displacement fields, masks, branch displacements and literal
offsets into the copied instruction bytes. The full list of substitution
codes is in [06-generated-code.md](06-generated-code.md#64-substitution-codes).
Special rows (`NV`, `ES`, `RE`, `HD`, `EN`, `ET`, lookups, writes, `CALL
VIEW`) have dedicated PASS1 routines because their skeletons include
prologue constants (`NVPROLOG`), lookup-buffer prefixes or end-of-set
branches back into `GVBMR95` (`MDLES`: `B EVNTPREV`).

`LTCODSEG` records the code-segment address for every row so that PASS2
(in `GVBMR95`) and the trace/snap facilities can find it, and `LTESCODE`
records the start of each `ES` set's code — the address `GVBMR95` branches
to for every event record.

## 4.12 OPENEXTF — extract files

`OPENEXTF` opens the extract files. Each `WR` row's `LTWREXTA` points at
an `EXTFILE` control area whose DCB is a copy of the `EXTRFILE` model
(`RECFM=VB, LRECL=8192`) with `DDNAME=EXTRnnn` — the prefix `EXTR` plus a
3- or 4-digit extract file number (`LTWREXT#`) built with `UNPK`. Files
with a number ≤ `MAXSTDF#` (the standard extract file count from the VDP
0801 record, also saved as `LTMAXFIL` in the `HD` row; forced to zero in the
reference phase) must be in the JCL; a view with a write exit (`LTWRNAME`) needs no DD at all. For
any other missing DD, `TREAT_MISSING_VIEW_OUTPUTS_AS_DUMMY=Y` (`EXECDUMY`)
makes `OPENEXTF` build a DCB named `DUMYnnn` that is never `OPEN`ed and
whose writes are discarded; with the default `N` the condition is an
`OPENERR3` error. Files used as pipes (`LTPIPELS`) get `PIPEBUFR` in-storage
buffers and `PIPECHK` check routines instead of BSAM buffers, and
`PAGE_FIX_IO_BUFFERS=Y` (`execpagf`) sets `DCBEBFXU` so that the buffers are
page-fixed.

## 4.13 Control report and error handling

`PRNTRPT` opens `EXTRRPT` (DCB `CTRLFILE`, `RECFM=VB,LRECL=164`) and
prints the parameter echo, environment variables and, at the end,
per-view/per-file statistics. Trace output goes to `EXTRTRAC` (`RECFM=FBA,
LRECL=161`). Errors are reported through the `GVBMSG` macro, which builds a
message from `GVBMSGDF`/`GVBMSGGE` definitions and either `WTO`s it or
writes it to the log; `ABEND_ON_ERROR_CONDITION=Y` causes `GVBUT99` to
abend instead of returning a non-zero code. `GVBMR96` returns to
`GVBMR95` with R15 = 0 on success; any non-zero value makes `GVBMR95`
skip thread attachment and end with that return code.
