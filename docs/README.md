# GenevaERS Performance Engine — Technical Documentation

This directory documents the **GenevaERS Performance Engine** (PE): a z/OS batch
extraction, transformation and reporting engine written almost entirely in
IBM High Level Assembler (HLASM). Everything here was derived by reading the
assembler sources in `ASM/`, the macros/DSECTs in `MAC/`, the build metadata in
`TABLE/`, `LINKPARM/`, `JCL/` and `zapp.yaml`, and the sample view definition
`V10727.xml`. Where a statement is an inference rather than something stated
in source comments or code, it is marked *(inferred)*.

The repository contains roughly 79,000 lines of HLASM across 24 programs and
92 macros. The two largest members, `GVBMR95.asm` (~28k lines) and
`GVBMR96.asm` (~19k lines), together implement a *runtime machine-code
generator*: the "logic table" produced by the GenevaERS workbench is compiled
at job start into real z/Architecture instructions, which are then executed
once per input record. Understanding that mechanism is the key to
understanding the rest of the code base.

## Reading order

| # | Document | What it covers |
|---|----------|----------------|
| 1 | [01-overview.md](01-overview.md) | What PE is, the four processing phases, repository layout, terminology primer |
| 2 | [02-architecture.md](02-architecture.md) | Component map, data flow between phases, DDNAME contracts, register conventions |
| 3 | [03-gvbmr95-extract-engine.md](03-gvbmr95-extract-engine.md) | `GVBMR95`: alias enforcement, storage, thread attach, EVENTS wait loop, event-record loop, generated-code dispatch, extract write, end-of-job |
| 4 | [04-gvbmr96-initialization.md](04-gvbmr96-initialization.md) | `GVBMR96`: parameter/VDP/logic-table loading, cloning, lookup loading, thread build, literal pools, PASS1 code generation, extract file open |
| 5 | [05-logic-table.md](05-logic-table.md) | Logic-table input record formats (`HD`,`NV`,`F0`,`F1`,`F2`,`RE`,`WR`,`CC`,...), the in-memory `LOGICTBL` DSECT, function-code catalogue |
| 6 | [06-generated-code.md](06-generated-code.md) | The runtime code generator: `FUNCTBL`, the `GVBmaj_t` major-function table and its 4x13x13 format arrays, model-code skeletons, `CS*` substitution codes, literal-pool header, PASS1/PASS2 relocation |
| 7 | [07-lookups-and-reference-data.md](07-lookups-and-reference-data.md) | Reference phase output (REH/RED), `LKUPBUFR`/`LKUPTBL`, `GVBSRCHR` binary search generator, hash lookups, effective dating |
| 8 | [08-threading-ziip-recovery.md](08-threading-ziip-recovery.md) | Sub-task model, work-unit queues (`PICKEVNT`), `EVENTS`, WLM enclave and zIIP SRB/TCB switching, ESTAE recovery |
| 9 | [09-io-handlers.md](09-io-handlers.md) | Event-file access methods: `GVBMRBS` (BSAM/pipes/tokens), `GVBMRVK` (VSAM), `GVBMRSQ` (DB2 SQL), `GVBMRSU`/`GVBMRHPU` (DB2 HPU), `GVBMRAD` (Adabas), plus `GVBUR20`, `GVBTP90` |
| 10 | [10-format-phase-gvbmr87-gvbmr88.md](10-format-phase-gvbmr87-gvbmr88.md) | `GVBMR87` (format-phase init) and `GVBMR88` (sorted extract → reports/files) |
| 11 | [11-utilities.md](11-utilities.md) | `GVBDL96`, `GVBDAYS`, `GVBUR33`, `GVBUR35`, `GVBUR39`, `GVBURALI`, `GVBURZTM`, `GVBUT99`, `GVBUTHDR`, `GVBUTMSG`/`GVBUTMUE` |
| 12 | [12-control-blocks.md](12-control-blocks.md) | DSECT reference: `THRDAREA`, `GENPARM`/`GENENV`/`GENFILE`, `EXECDATA`, `EXTFILE`, `EXTREC`, VDP record layouts, macro inventory |
| 13 | [13-parameters-ddnames-messages.md](13-parameters-ddnames-messages.md) | `EXTRPARM`/`MR88PARM` keywords, trace parameters, DDNAME table, return codes, message catalogue structure |
| 14 | [14-build-and-deployment.md](14-build-and-deployment.md) | Encoding (`.gitattributes`), `zapp.yaml`, `LINKPARM/*.ftl` link-edit templates, aliases, DB2 bind JCL |
| 15 | [15-glossary.md](15-glossary.md) | Domain and z/OS vocabulary |

## One-paragraph summary

A GenevaERS "view" is designed in a workbench and compiled into two artefacts:
a **VDP** (View Definition Parameters — binary records describing files,
logical records, fields, lookups, columns, sort keys, titles) and a **logic
table** (row-oriented pseudo-instructions: read event file, test field, build
lookup key, do lookup, move column, write extract, ...). At run time
`GVBMR95` loads `GVBMR96`, which reads both artefacts, validates every row
against a function-code table, allocates and loads reference (lookup) tables
into memory, and translates each row into z/Architecture machine code by
copying *model code* skeletons and patching in offsets, lengths, addresses and
branch displacements. `GVBMR95` then attaches one z/OS sub-task per event file
(or per work unit), each of which reads its event file with the appropriate
access method and branches into the generated code for every record. Extract
records are written to `EXTRnnn` files; the same executable, run under alias
`GVBMR95R`, produces reference-table files (`REH`/`RED`) for later lookups.
After an external sort, `GVBMR88` (initialised by `GVBMR87`) reads the sorted
extract, detects sort-key breaks, accumulates subtotals, formats columns with
`GVBDL96`, and writes reports, CSV, XML or fixed-format output files.
