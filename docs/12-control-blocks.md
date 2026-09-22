# 12. Control blocks (DSECT catalogue)

This chapter is the map of the storage structures the Performance Engine
uses. Each entry gives the macro that defines the DSECT, who owns the storage,
the fields you will need when reading the assembler, and a pointer to the
chapter where the behaviour is explained. Field names are exactly as they
appear in `MAC/*.mac`; when only a subset of a DSECT is listed the omission is
marked with `...`.

```text
                          storage ownership at run time

  GVBMR95 mother TCB                     daughter TCB / SRB (one per thread)
  ┌──────────────────────────────┐      ┌──────────────────────────────┐
  │ THRDAREA (main)  R13 ───────►│      │ THRDAREA (thread)  R13 ─────►│
  │  ├ EXECDATA   (EXECDADR)     │      │  ├ THRDMAIN ──► main THRDAREA│
  │  ├ PARMTBL    (PARMTBLA)     │      │  ├ THRDRE / THRDES  (rows)   │
  │  ├ VDP table  (VDPBEGIN)     │      │  ├ CODEBEG..CODEEND          │
  │  ├ LOGICTBL   (LTBEGIN)      │      │  ├ THRDLITP ──► LITP_HDR ... │
  │  ├ EXTFILE[]  (EXTFILEA)     │      │  ├ env_area   (GENENV)       │
  │  ├ TBLHEADR[] (rehtbla)      │      │  ├ file_area  (GENFILE)      │
  │  ├ LKUPBUFR/LKUPTBL pools    │      │  ├ PARM_AREA  (GENPARM)      │
  │  └ EVENTS ECB list           │      │  └ SAVEAREA/SAVESUBR/...     │
  └──────────────────────────────┘      └──────────────────────────────┘
```

## 12.1 Where the DSECTs live

| Macro | DSECTs defined | Used by |
|-------|----------------|---------|
| `MAC/GVBMR95W.mac` | `THRDAREA`, `dfptrace`, `runview_list` | `GVBMR95`, `GVBMR96` |
| `MAC/GVBMR95C.mac` | `EXTFILE`, `EXHEXH`, `EXUEXU`, `LTWRAREA`, `LKUPBUFR`, `LKUPTBL`, `TBLHEADR`, `FUNCTBL`, `NVPROLOG`, `callview_dsect`, `LITP_HDR`, `lpstatlitp`, `COLEXTR`, `ENVVTBL`, `PARMTBL`, `LEINTER`, `SYMTABLE`, `SUMAREA`, `STACKENT`, `EXTREC`, `INITVAR`, `HASH_LU_ENT`, `exit_data`, `DECB` | `GVBMR95`, `GVBMR96`, I/O drivers |
| `MAC/GVBMR95L.mac` | `LOGICTBL` and its 12 `ORG LTREDEFN` function-specific overlays | `GVBMR95`, `GVBMR96`, `GVBMR87`, `GVBMR88` |
| `MAC/GVBX95PA.mac` | `GENPARM`, `GENENV`, `GENFILE`, `GENSTART`, `GENEVENT`, `GENEXTR`, `GENKEY`, `GENWORK`, `GENRTNC`, `GENBLOCK`, `GENBLKSZ` | `GVBMR95` and every read/lookup/write exit and I/O driver |
| `MAC/EXECDATA.mac` | `EXECDATA` | `GVBMR95`, `GVBMR96` (parsed `EXEC PARM`/`MR95PARM`) |
| `MAC/GVBMR88W.mac` | `WORKAREA` (format phase) | `GVBMR87`, `GVBMR88` |
| `MAC/GVBMR88C.mac` | `FLDDEFN`, `VIEWREC`, `SORTKEY`, `COLDEFN`, `CALCTBL`, `RTITLE`, `EXTREC`, `HDRREC`, `HDRDATA`, `CTLREC`, `COLEXTR`, `LKUPBUFR`, `LKUPTBL`, `TBLHEADR`, `LEINTER`, `PARMDATA`, `ENVVTBL` | `GVBMR87`, `GVBMR88` |
| `MAC/GVBAX88P.mac` | `GENPARM`, `GENRPT`, `GENRUN` | format-phase exits |
| `MAC/VDPHEADR.mac`, `MAC/GVBnnnnA.mac`, `MAC/GVBnnnnB.mac` | `vdp_header`, one `VDPnnnn_*_RECORD` per VDP record type | all phases |
| `MAC/GVBLT*.mac` | `LTHD_REC`, `LTF0_REC`, `LTF1_REC`, `LTF2_REC`, `LTNV_REC`, `LTRE_REC`, `LTWR_REC`, `LTCC_REC`, `LTV1_REC`, `LTV2_REC`, `LTVN_REC`, `LTVV_REC`, `LTGN_REC`, `LTBLV3` | `GVBMR96` (logic-table load) |
| `MAC/GVBAUR35.mac` | `M35SVC99` | `GVBUR35`, `GVBMR95` |
| `MAC/GVBHDR.mac` | `Headerpr` | `GVBUTHDR` callers |
| `MAC/DL96AREA.mac` | `DL96AREA` | `GVBDL96` |
| `MAC/REPTDATA.mac` | `irunrept`, `irefrept`, `sortrept`, `isrcrept`, `ofilrept`, `execrept` | control-report builders |
| `MAC/MAJRFTAB.mac` | `major_func_table` | `GVBMR96` function matching |
| `MAC/GVBPARM.mac` | `PARMLIST` | `GVBMR88` `PARM=` parsing |

The remaining `MAC/*.mac` files are code-generating macros (e.g. `GVBMSG`,
`GVBSRCH`, `LKUPCODE`, `GVBLOGIT`, `GVBPHEAD`) or `EQU` sets
(`GVBUTEQU`, `DL96EQU`, `GVBMRZPE`) rather than DSECTs.

## 12.2 `THRDAREA` – the thread work area

Defined in `MAC/GVBMR95W.mac`; length `THRDLEN`. There is one instance per
thread plus one for the mother task, and **R13 always points at the current
thread's `THRDAREA`** in `GVBMR95`, `GVBMR96` and the generated code. The
first `18fd` doubleword save areas are what makes `STMG/LMG` linkage into the
drivers possible without a separate stack.

### Save areas

| Field | Purpose (source comment) |
|-------|--------------------------|
| `SAVEAREA DS 18fd` | register save area (standard 144-byte F4SA-style) |
| `SAVESUBR DS 18fd` | event-read save area (used by driver calls) |
| `SAVEZIIP DS 18fd` | zIIP TCB/SRB switch save area |
| `SAVESUB2`, `SAVESUB3 DS 18fd` | second/third-level subroutine save areas |
| `cookie_save DS 18fd`, `cdd_save DS 4fd` | cookie and `Conv_date_daynum` routines |
| `SAVETCB`, `SAVESRB DS 18fd` | save areas around the `IEA4XFR` pause/transfer calls |
| `save_grande DS 15fd`, `registers DS 16fd`, `estae_rsa DS 16f` | ESTAE / diagnostics |
| `fp_reg_savearea DS 8fd` | eight 64-bit floating-point registers (DFP work) |

### Thread linkage and scheduling

| Field | Purpose |
|-------|---------|
| `THRDMAIN` | address of the mother's `THRDAREA` (every thread can reach the shared counters) |
| `THRDFRST`, `THRDPREV`, `THRDNEXT` | doubly linked list of thread areas |
| `TCBADDR`, `TASKECB` | TCB address and ECB posted at thread end (see chapter 8) |
| `estae_stop` | non-zero once `STATUS STOP` has been issued by the recovery routine |
| `WAITECB DS 2A` | ECBs a thread waits on (pipe / token synchronisation) |
| `ENCLTOKN DS FD`, `WLMTOKEN` | WLM enclave token and period token |
| `SRB_active`, `SRB_Limit` | number of SRBs executing / configured limit (mother only) |
| `SRBTOKEN DS FD`, `Stoken` | SRB suspend/resume token, STOKEN |
| `Thread_mode DS C` | `'P'` problem state or `'S'` supervisor |
| `localziip`, `localauth`, `workazip` | copy of `EXECZIIP`, `'A'`/`'N'` authorised flag, address of `GVBMRZP` |
| `TCBPET1A/2A`, `TCBPET1/2`, `SRBPET1A/2A`, `SRBPET1/2` | pause-element tokens for TCB and SRB sides (`MUST stay together`) |
| `XFRTCBPL`, `XFRSRBPL DS 7 Ad` | `IEA4XFR` parameter lists |
| `ZIIPQUALTIME`, `ZIIPONCPTIME`, `ZIIPTIME`, `enc_cputime DS FD` | accumulated timings passed to `GVBURZTM` |
| `Thread_fail` | failing thread ID set by `TASKABND` via CS |
| `THRDTYP DS HL2` | `DISKDEV`/`TAPEDEV`/other – drives `PICKEVNT` queue choice |
| `THRDCNT`, `THRDDONE`, `VIEWCNT` | counts across all threads |

### Table anchors (mother task; threads read through `THRDMAIN`)

| Field | Points to |
|-------|-----------|
| `EXECDADR` | `EXECDATA` (parsed `EXEC PARM`) |
| `PARMTBLA` | first `PARMTBL` entry (`MR95PARM` trace/abend parameters) |
| `VDPBEGIN`, `VDPCOUNT`, `VDPSIZE`, `vdp_addr_curr_seg`, `vdp_seg_len`, `vdp_seg_cnt` | in-memory VDP (segmented) |
| `LTBEGIN`, `LTCOUNT`, `LTSIZE`, `LTROWADR` | `LOGICTBL` rows and the row-number → address table |
| `LTHDROWA`, `LTENROWA` | `HD` and `EN` rows |
| `LTNXDISK`, `LTNXTAPE`, `LTNXOTHR` | lock-free work queues (chapter 3.11) |
| `THRDEXEC` | ES sets executed by this thread |
| `EXTFILEA`, `MAXSTDF#`, `MAXEXTF#`, `filecnt_real` | `EXTFILE` array |
| `rehtbla`, `rehcnt`, `grefcnt` | `TBLHEADR` array (chapter 7) |
| `refpools`, `refpools_real`, `refpoolb`, `refpoolc DS fd` | reference-data pool sizing and cursor |
| `GLOBVNAM` | global variable name list |
| `vdp0650a`, `vdp0801a` | in-memory copies of the join and extract-file-count records |
| `runview_ptr` | `runview_list` when `RUNVIEWS` was specified |
| `EXIT_DATA_PTR` | chain of `exit_data` blocks |
| `CTRLDCBA`, `Logfdcba`, `TRACDCBA`, `HASHDCBA`, `snapdcba`, `hdrdcba`, `ltbldcba` | report/log/trace/snap DCBs |

### Generated code and literal pools (per thread)

| Field | Purpose |
|-------|---------|
| `CODESIZE`, `CODEBEG`, `CODEEND` | this thread's generated-code buffer |
| `THRDLITP` | this thread's first `LITP_HDR` |
| `THRDRE`, `THRDES` | the `RE`/`ES` rows currently executing |
| `litpcnt`, `litpool_sz`, `litptots`/`Tl*` fields | literal-pool sizing statistics for the control report |
| `SAVEWR`, `SAVER10`, `SAVEPOS`, `lt_saver9`, `default_func` | temporary registers/state for generated-code helpers |
| `ovflmask` | value ORed into the PSW program mask when `OVERFLOW=ON` |
| `dfp_quantum_ptr`, `printdfp_save`, `print_dfp1/2` | DFP support |

### Event-file state (per thread, filled by the I/O driver)

| Field | Purpose |
|-------|---------|
| `env_area`, `file_area`, `PARM_AREA` | inline `GENENV`, `GENFILE`, `GENPARM` passed to drivers/exits |
| `EVNTSUBR DS CL8`, `EVNTREAD DS A` | driver name and read entry point |
| `EVNTDCBA`, `EVNTGETA`, `EVNTCHKA`, `EVNTDECB`, `EVNTdecbp`, `EVNTbufp`, `EVNTbufps`, `EVNTbufno` | DCB, GET/CHECK routines, DECB/buffer pools |
| `evntpgfs`, `evntpgfe` | page-fix range when `PAGE_FIX=Y` |
| `RECADDR`, `RECEND`, `EODADDR`, `PREVRECA DS ad` | 64-bit current record / end / end-of-data / previous record |
| `EOFEVNT DS C` | end-of-file encountered |
| `RETNCODE`, `RETNPTR DS ad`, `RETNBSIZ` | return code / block pointer / block size from an exit |
| `LKUPKEY DS XL256` (overlays `WKREENT DS 0XL256`) | lookup key work area (also the reentrant parameter area) |
| `LBCHAIN DS 0CL6` | anchor of the lookup-buffer chain |
| `TP90LIST`/`TP90AREA` (`PAANCHOR`, `PADDNAME`, `PAFUNC`, `PAFTYPE`, `PAFMODE`, `PARTNC`, `PAVSAMRC`, `PARECLEN`, `PARECFMT`, `PARESDS`) | `GVBTP90` parameter list and file-spec entry |
| `DL96LIST` (`DL96PA`, `DL96TGTA`, `DL96LENA`, `DL96RTNA`) | `GVBDL96` parameter list |
| `DYNAAREA DS (M35S99LN)C` | `GVBUR35` parameter block |
| DB2 block: `DSNALI`, `DSNHLI2`, `SQLDADDR`, `SQLTADDR`, `ROWLEN`, `ROWADDR`, `DBSUBSYS`, `DB2PLAN`, `SQLCA` (`SQLCODE`, `SQLERRM`, `SQLSTATE`, ...) | shared with `GVBMRSQ`/`GVBMRDV` |
| `vs_*` fields (`vs_gplwork`, `vs_clusname`, `vs_locbuf`, ...) | SVC 26 catalogue lookup used to find the VSAM data component (`@02I`) |

### Statistics and reporting

| Field | Purpose |
|-------|---------|
| `thrd_evntbyte_cnt`, `thrd_evntrec_cnt`, `thrd_extrbyte_cnt`, `thrd_extrrec_cnt`, `grand_total_lkups DS xl8` | per-thread 64-bit counters |
| `event_file_cnt`, `event_pipe_token_cnt`, `event_database_cnt`, `event_read_exit_cnt` and matching `*_rec_cnt`/`*_byt_cnt DS xl8` | run totals by event-source class (mother) |
| `extr_file_cnt`, `extr_pipe_token_cnt`, `extr_rec_disk_cnt`, `extr_rec_pitk_cnt`, `extr_byt_*` | extract totals |
| `PRNTBUFF DS 0CL164` ... `PRNTLINE`, `PRNTCC`, `PRNTTEXT`, `PRNTCNT`, `PRNTFILE`, `PRNTFND`, `PRNTNOT`, `PRNTSTAT` | control-report line (RDW + 161 bytes) |
| `errb DS 0CL130` (`errbrdw`, `error_bufl`, `error_buffer DS CL126`) | `GVBMSG` output buffer (126 = WTO maximum) |
| `HDRREC DS 0CL16` (`HDRECLEN`, `HDSORTLN`, `HDTITLLN`, `HDDATALN`, `HDNCOL`, `HDVIEW#`) + `HDRDATA DS 0CL31` (`HDRECCNT`, `HDUSERID`, `HDEVNTNM`, `HDSATIND`, `HD0C7IND`, `HDOVRIND`, `HDLIMIT`) | extract-file header/control record written at end of each view |
| `return_code`, `reason_code`, `error_address`, `overall_return_code`, `print_Return_code` | run outcome |
| `NAMEPGM DS cl8`, `namePGMl`, `alias equ *,x'80'` | alias name from `GVBURALI` and the alias flag |
| `WORKFLAG1`: `DB2_USED x'80'`, `msg811done x'40'`, `MSGLVL_DEBUG x'20'` | run flags |
| `format_phase`, `extract_phase DS c` | phase indicators |
| `VDP_DATE`, `VDP_TIME`, `VDP_desc`, `logictbl_date`, `logictbl_time`, `logictbl_desc` | headers echoed in the control report and used for the timestamp check |
| `pgmwork DS cl1000` | scratch area available to helper routines |

## 12.3 `EXECDATA` – parsed execution parameters

`MAC/EXECDATA.mac`. Built by `GVBMR96` from the `MR95PARM`/`EXEC PARM`
keywords (chapter 13 lists the keyword → field mapping). All fields are
character except `EXECVERS` and `EXEC_HASHMULTB`.

```asm
EXECDATA DSECT
EXECVERS DS    FL04            logic table version no
EXECMSDN DS    CL03            MULTSDN
         ORG   EXECMSDN
EXECNRD  DS    CL02            number of read buffers
EXECNWRT DS    CL02            number of write buffers
EXECSNGL DS    CL01            first/single thread mode ('1','A','N')
EXECTRAC DS    CL01            trace on/off
EXECSNAP DS    CL01            snap (logic table + code) on/off
EXECVSIZ DS    CL06            VDP table size (*1024)
EXECDISK DS    CL04            number of disk threads
EXECTAPE DS    CL04            number of tape threads
EXECRLIM DS    CL13            event-file read limit
EXECDUMY DS    CL01            dummy extract files not in JCL
EXECSPLN DS    CL08            DB2 plan override - SQL  (GVBMRSQ)
EXECVPLN DS    CL08            DB2 plan override - VSAM (GVBMRDV)
EXECZIIP DS    CL01            zIIP requested (Y)
EXEC_SRBLIMIT ds cl4           maximum number of active SRBs
execovfl_on ds cl1             PSW overflow mask on
execpagf    ds cl1             page fixing allowed Y/N
exec_bdate  ds cl8             BATCH_DATE
exec_rdate  ds cl8             RUN_DATE
exec_fdate  ds cl8             FISCAL_DATE
EXECTPLN DS    CL08            DB2 plan override (MRCT)
EXECmsgab   ds cl5             abend on message number
EXECltab    ds cl8             abend at logic table row
EXEC_db2_df ds  cl03           DB2 VSAM date format
exec_optpo  ds  c              optimize packed output Y/N
exec_uabend ds  c              user abend (or return code) Y/N
exec_estae  ds  c              ESTAE enabled (default Y)
EXEC_Dump_Ref DS CL1           include ref pools in dump Y/N (default Y)
EXEC_check_timestamp DS CL1    check VDP & JLT/XLT timestamps match Y/N
EXEC_LOGLVL ds  CL8            EXTRLOG/REFRLOG message level
EXEC_HASHPACK DS CL1           pack key before CHECKSUM in hash
EXEC_DISPHASH DS CL1           display hash stats
EXEC_HASHMULT ds CL2           hash table size multiplier
EXEC_HASHMULTB ds CL4          ... in binary
EXEC_DB2HPU    ds CL1          DB2 HPU utility
EXECDLEN EQU   *-EXECDATA
```

## 12.4 `PARMTBL` – `MR95PARM` trace/abend parameter entries

`MAC/GVBMR95C.mac`. One linked entry per `MR95PARM` card that carries
view-specific trace or abend criteria (chapter 13, `TRACE=`).

```asm
PARMTBL  DSECT
PARMNEXT DS    AL4          next entry
PARMVIEW DS    FL4          view id
PARMDDN  DS    CL8          event file DDNAME
PARMFROM DS    xl8          beginning event record count
PARMTHRU DS    xl8          ending event record count
PARMROWF DS    FL4          trace logic table row - from
PARMROWT DS    FL4          trace logic table row - thru
PARMFLEN DS    FL4          trace function code length
PARMFUNC DS    CL4          trace function code
PARMLTAB DS    FL4          abend logic table row no.
PARMSGAB DS    FL4          abend message no.
PARMVOFF DS    HL2          event field value offset
PARMVLEN DS    HL2          event field value length
PARMVALU DS    CL16         event field value
PARMVALD DS    CL36         event field value - displayable
PARMDUMP DS    CL8          include event record hex dump in trace
```

## 12.5 `LOGICTBL` and the `LTREDEFN` overlays

The common prefix is described in chapter 5.2. `MAC/GVBMR95L.mac` then
issues `ORG LTREDEFN` twelve times to overlay function-specific fields. The
overlays that other chapters refer to:

| Overlay (representative fields) | Rows | Where used |
|--------------------------------|------|-----------|
| `LTFILTYP`, `LTFILEDD`/`LTFILEID`, `LTREPFcnt`, `ltre_next_exit`, `LTREEXIT`/`LTRENAME`/`LTREADDR`/`LTREENTP`/`LTREWORK`/`LTREPARM`, `LTREES`, `ltreltpl`, `ltrertkn_parent`, `ltreindx`, `ltrerecl` | `RE`, `RETK`, `RETX` | chapters 3, 9 |
| `LTESSET#`, `LTTHRDWK`, `LTESVNAM`, `LTPIPELS`, `LTNEXTES`, `LTFRSTRE`, `LTFRSTNV`, `LTTOKNNV`, `LTSAMES#`, `LTCLONES`, `LTESINIT`, `LTESCODE`, `LTESLPSZ`, `LTESLPAD`, `ltesrtkq`, `ltesretn`, `ltespr11`, `ltes_time` | `ES` | chapters 3, 6 |
| `LTDATALN`, `LTMAXCOL`, `LTMAXOCC`, `LTSUMCNT`, `LT1000A`, `LTNVVNAM`, `LTNVTOKN`, `LTNEXTNV`, `LTFRSTLU`, `LTFRSTWR`, `LTVIEWRE`, `LTVIEWES`, `LTPARENT`, `LTPARMTB`, `LTSUBOPT`, `LTNVRELO` | `NV` | chapter 6 |
| `LTWREXT#`, `LTWRFILE`/`LTWRFID`, `LTWR200A`, `LTWREXIT`/`LTWRNAME`/`LTWRADDR`/`LTWRENTP`/`LTWR_Workarea`/`LTWRPARM`, `LTWRLUBO`, `LTWREXTA`, `LTWRDEST`, `LTWRSUMC`, `LTWRLMT`, `LTWRNEXT`, `LTWRVIEW`, `ltwrtkrc`, `ltwrre`, `LTWRPGM_TYPE`, `LTWR_offset_ct` | `WR*` | chapters 3.10, 6 |
| `LTCOLNO`, `LTRUNDT`, `LTCCLEN1`, `LTERRLEN`, `LTOVRLEN`, `LTRELOPR`, `LTJUSOFF` | `CC`, `DT*`, `CT*`, `SK*`, `KS*`, `CF*` | chapter 6 |
| `LTLUFILE`/`LTLUFID`, `LTLULRID`, `LTLUPATH`, `LTLUEXIT`/`LTLUNAME`/`LTLUptyp`/`LTLUADDR`/`LTLUENTP`/`LTLUWORK`/`LTLUPARM`, `LTLUWPATH`, `LTLUopt`, `LTLUNEXT`, `LTLKUPOS`, `LTLUSTEP` | `LU*`, `LK*`, `JOIN`, `KS*` | chapter 7 |
| `LTFUNTBL`, `LTCODSEG`, `LTGENLEN`, `LTLUBOFF`/`LTWREXTO` (common prefix) | all | chapter 6 (PASS1/PASS2) |

The read, lookup and write exit descriptors are laid out so that
`ltre_next_exit`, the LU equivalent and `ltwr_next_exit` sit at the same
offset; the macro enforces this with
`ASSERT (ltwr_next_exit-logictbl),EQ,(ltre_next_exit-logictbl)`.
`LTWRPGM_TYPE` values: `LECOBOL X'01'`, `COBOL2 X'02'`, `C X'03'`,
`CPP X'04'`, `JAVA X'05'`, `ASM X'06'`.

Flag byte values (`LTFLAG1`/`LTFLAG2`) are listed in chapter 5.2.

## 12.6 `LITP_HDR` and `NVPROLOG` – per-view runtime state

`LITP_HDR` heads every view's literal pool; **R2** is the biased base of the
current pool at run time and `4(,R2)` is `lp_base_litp` (chapter 6.5).

```asm
LITP_HDR DSECT
LPVADDR  DS    AL04         current view code address
LPlurtnc DS    fL04         last lookup return code
lp_saved_litp ds al04       saved LITP_HDR (callview)
lp_return_adr ds al04       return address (callview)
lp_nvcons_adr ds al04       save for R11 if WRTK/WRTX goes to NV
LPEXTCNT DS    PL06         view extract records written
LPLKPFND DS    PL06         view lookups found
LPLKPNOT DS    PL06         view lookups not found
         DS    XL02
lp_prev_litpo  ds al04      offset of previous literal pool
lp_base_litp   ds al04      always the base literal pool address
lp_RE_addr     ds al04      address of RE entry
lp_r6_save     ds cl08      R6 save (event record address)
LITPHDRL EQU   *-LITP_HDR
```

`NVPROLOG` is the instruction skeleton copied in front of every view's code
(the first 16 bytes after the three instructions are data, not code):

```asm
NVPROLOG DSECT
NVNOP    jlnop *             NOP -> patched to a branch to disable the view
NVLAYR8  lay   r8,0(0,0)     initialise CT column occurs pointer
NVBRNCH  bras  r14,nvviewmv  branch around the header constants
NVCONST  DS   0XL16
NVVIEWID DS    FL04          view id
NVLOGTBL DS    AL04          NV row address
NVLITPSZ DS    FL04          literal pool size (this view)
NVNXVIEW DS    AL04          next view code address
NVPROLEN EQU   *-NVPROLOG
nvviewmv mvc   0(0,0),0(0)   move current view id
         lay   r15,0         lit pool base is 512k from start
         mvc   0(0,0),0(0)   copy NVLOGTBL to LITP_HDR
         la    r11,0(,0)     next view address -> R11
NVTRACE  jnop  *             branch to trace routine if enabled
NVLEVNT  LG    R6,0(0,0)     initialise event record base register
NVCODELN EQU   *-NVPROLOG
```

`callview_dsect` is the corresponding skeleton for `WRTK`/`WRTX` token calls;
it switches R2 to the called view's literal pool and stores the return address
at `LTESRETN` (source comments tagged `pgc99`/`pgc1`).

## 12.7 `EXTFILE` – extract-file control area

One entry per extract file number; `EXTFILEA` in `THRDAREA` addresses the
array and `MAXEXTF#` bounds it (chapter 3.10 and 4).

```asm
EXTFILE  DSECT
EXTDDNAM DS    CL8          DDNAME
EXTVDPA  DS    FDL08        VDP 0200 record address
EXTDCBA  DS    A            DCB
EXTDECBF DS    A            first DECB
EXTDECBC DS    A            current DECB
EXTPUTA  DS    A            write routine
EXTCHKA  DS    A            check routine
EXTPRINT DS    A            print-next pointer
EXTINUSE DS    A            in-use pointer
EXTEOBAD DS    A            current end-of-buffer
EXTRECAD DS    A            current record address
EXTRECLN DS    HL2          current record length
EXTCNT   DS    XL8          record count (binary)
EXTBYTEC DS    xL8          byte count (binary)
EXTRECFM DS    XL2          record format
EXTMINLN DS    HL2          minimum record length
EXTLRECL DS    HL2          LRECL
EXTPIPEP DS    HL2          parent thread count ("piped input")
EXTPIPED DS    HL2          thread done count
EXTPIPEC DS    HL2          child thread count
EXTBUFNO DS    HL2          number of buffers
EXTBLKSI DS    HL2          BLKSIZE
extflag  ds    x            extfmtph x'80' = DDNAME used in a format phase
EXTPUT_6431  DS A           AMODE 64 write routine
EXTCHK_6431  DS A           AMODE 64 check routine
EXTFILEL EQU   *-EXTFILE
```

The `EXTPIPE*` counters implement pipe synchronisation between writer
threads and the `GVBMRBS` pipe reader (chapter 9 section 2).

## 12.8 `EXTREC` – the extract record

Two definitions exist: `MAC/GVBMR95C.mac` (writer side) and
`MAC/GVBMR88C.mac` (reader side). The prefix is identical and matches the
`GENEXTR` view of the same bytes given to exits:

```text
offset  GVBMR95C     GVBX95PA (GENEXTR)         meaning
   0    EXRECLEN     GP_EXT_REC_LENGTH          RDW length
   2    (XL02)       (XL2)                      RDW flags
   4    EXSORTLN     GP_SORT_KEY_LENGTH         sort key length
   6    EXTITLLN     GP_TITLE_KEY_LENGTH        sort title length
   8    EXDATALN     GP_DATA_AREA_LENGTH        extract data length
  10    EXNCOL       GP_NBR_CT_COLS             number of CT columns
  12    EXVIEW#      GP_VIEW_ID                 view id (+X'80000000')
  16    EXSRTKEY     GP_EXTRACT_VAR_LEN_AREA    sort key, title, data, CT cols
```

The `EXVIEW#` high-order bit (`+X'80000000'`) marks a data record; the
header/control record written at view end uses `HDRREC` (`HDVIEW#` without
the bit) followed by `HDRDATA`. `CT` column values are appended as `COLEXTR`
entries (`COLNO DS HL02`, `COLDATA DS PL12`). The `ORG EXTREC` / `BILLREC`
overlay in `GVBMR95C.mac` is a legacy 80-byte billing record layout.

## 12.9 `LTWRAREA`, `SUMAREA`, `STACKENT` – extract-summarisation state

`WRSU` (write summarised) rows point through `LTWRAREA` to a `SUMAREA`:

```asm
LTWRAREA DSECT
LTWRROWA DS    AL04         logic table row
LTWRWORK DS    AL04         write exit work-area anchor
LTWRSUMA DS    AL04         summary view work area (SUMAREA)
LTWRCNTI DS    xl08         extract record count - input
LTWRCNTO DS    xl08         extract record count - output
```

`SUMAREA` holds a 64-bit (`IARV64`) hash-indexed stack of extract records;
`STACKENT` is the 48-byte prefix of every stacked record:

```asm
SUMAREA  DSECT                    STACKENT DSECT
SUMSAVE  DS  18FD                 STKPREV  DS FD   previous entry
STAKBEG/STAKEND/STAKTOP/STAKBOT   STKNEXT  DS FD   next entry
STAKCURR DS FD                    STKHASHA DS FD   hash anchor
HASHBEG  DS FD                    STKSYPRV DS FD   synonym chain prev
ANCRCURR DS FD                    STKSYNXT DS FD   synonym chain next
STAKHDR  DS FD                    STKACUM  DS FD   CT column accumulators
SUMTEMP  DS FD                    STKLENG  DS H    record length w/o CT cols
SCALE    DS FD  hash key scale    STKMAXC  DS H    max column number used
PRIME    DS FD  hash key prime    STKRDW   DS 0H   RDW of saved extract record
SUMMINC/SUMMAXC DS H              STACKLEN EQU *-STACKENT
ELEMSIZE DS F
MAXSYNLN DS F   max synonym chain
         IARV64 MF=(L,GETMAIN)
SUMEXTR  DS 0C
```

`EXTSUM64` in `THRDAREA` accumulates the megabytes obtained for these
above-the-bar areas (the summarization `IARV64 GETSTOR` in `GVBMR95` adds
its rounded size to `EXTSUM64` and fails with `EPA_LIMIT` when a single
request exceeds 20 GB); `extest64` is declared next to it but no statement
in this tree stores into it. The size is derived from the view's
summarization settings in the VDP, not from an execution parameter.

## 12.10 Lookup structures

`TBLHEADR`, `LKUPBUFR`, `LKUPTBL` and `HASH_LU_ENT` are explained in
chapter 7. `HASH_LU_ENT` is the per-`LF/LR` hash configuration parsed from
`MR95PARM HASH=`:

```asm
HASH_LU_ENT DSECT
HASH_NEXT  DS  A        next in chain
HASH_LF    DS  F        LF id
HASH_LR    DS  F        LR id
HASH_MULT  DS  F        multiplier 1-10
HASH_PACK  DS  C        pack the key before CKSM, Y/N
```

## 12.11 Exit and driver interface blocks (`MAC/GVBX95PA.mac`)

`GENPARM` is the R1 parameter list handed to every event-read driver,
read/lookup/write exit and to `GENWRITE`. The `THRDAREA` fields `PARM_AREA`,
`env_area` and `file_area` are the inline instances.

```asm
GENPARM  DSECT                  GENENV   DSECT
GPENVA   DS A  -> GENENV        GPTHRDNO DS H     thread number
GPFILEA  DS A  -> GENFILE       GPPHASE  DS CL02  execution phase
GPSTARTA DS A  -> GENSTART      GPVIEW#  DS FL04  current view
GPEVENTA DS A  -> GENEVENT      GPENVVA  DS AL04  ENVVTBL address
GPEXTRA  DS A  -> GENEXTR       GPJSTPCT DS FL04  join step count
GPKEYA   DS A  -> GENKEY        GPJSTKA  DS AL04  join stack address
GPWORKA  DS A  -> GENWORK       GP_PROCESS_DATE  DS XL8
GPRTNCA  DS A  -> GENRTNC       GP_PROCESS_TIME  DS XL8
GPBLOCKA DS A  -> GENBLOCK      GP_ERROR_REASON     DS FL04
GPBLKSIZ DS 0A -> GENBLKSZ      GP_ERROR_BUFFER_PTR DS AL04
GENPARM1..5 DS A                GP_ERROR_BUFFER_LEN DS FL04
GENPARM_L EQU *-GENPARM         GP_PF_count DS XL04  PFs in current LF
                                GP_THRD_WA  DS AL04  current THRDAREA
```

```asm
GENFILE  DSECT
GPDDNAME DS    CL08     event file DDNAME (must be first)
GPRECCNT DS    xl08     record count (8-byte binary)
GPRECFMT DS    CL01     F, V, D
GPRECDEL DS    CL01     record delimiter
GPRECLEN DS    FL04     current record length
GPRECMAX DS    FL04     maximum record length
GPBLKMAX DS    FL04     maximum block size
GPLFID   DS    FL04     associated LF id
gp_call_srb ds C        Y = driver may be called in SRB mode
GP_redrive  ds C        Y = switch to TCB and redrive the read
```

The remaining single-field DSECTs are the targets of the `GENPARM` pointers:
`GENSTART` (`GP_STARTUP_DATA CL32`), `GENEVENT` (`GP_EVENT_REC AD`),
`GENEXTR` (see 12.8), `GENKEY` (`GP_LOOKUP_KEY CL256`), `GENWORK`
(`GP_WORK_AREA_ANCHOR A`), `GENRTNC` (`GP_RETURN_CODE A`), `GENBLOCK`
(`GP_RESULT_PTR AD`), `GENBLKSZ` (`GP_RESULT_BLK_SIZE F`). Note that the event
record and result pointers are **8-byte** fields: drivers store 64-bit
addresses even though the shipped drivers run AMODE 31.

`exit_data` describes each loaded exit program:

```asm
exit_data dsect
exit_next  ds a       next exit area
exit_Id    DS F       exit id (VDP 0210)
exit_NAME  DS CL08    exit name
exit_addr  DS A       load address
exit_entry ds a       entry point
exit_WORK  DS A       exit work-area anchor
exit_parm  DS CL32    start-up parameters
```

`LEINTER` is the Language Environment pre-initialisation (`CEEPIPI`) block
used when an exit is an LE-enabled COBOL/C program: function code, entry,
LE token, parameter list, return/reason codes, feedback area, DECB lists for
`GENPIPE`/`GENREAD`, and the `LELRECL`/`LERECFM` values returned by a read
exit.

## 12.12 `EXHEXH` / `EXUEXU` – DB2 HPU exit table

Shared between `GVBMRSU` and `GVBMRHPU` (chapter 9 section 4):

```asm
EXHEXH   DSECT              table header
EXHEYE   DS    CL8
EXHPARLL DS    H            number of table entries (parallel unloads)
EXHINIT  DS    H            source initialisations
EXHFINI  DS    H            source terminations

EXUEXU   DSECT              one entry per INZEXIT instance
EXUOUTBN DS    A            association to the HPU sub-task
EXUASSOC DS    XL1          association done
EXUECBMA DS    XL4          ECB the main task waits on
EXUECBEX DS    XL4          ECB the exit waits on
EXURPOS  DS    A            current record position in block
EXURLAST DS    A            last byte in data block
EXUROWLN DS    F            calculated row length
EXUCNT1  DS    F            times this instance returned a block
EXUEOF   DS    X            final block returned
EXUWAIT  DS    X            buffer filled, waiting
EXUSTAT  DS    X            1 = filling buffer
EXUFINAL DS    X            pending final call
EXURNUM  DS    F            records in returned block
EXUBLKA  DS    A            data block address
```

## 12.13 `INITVAR` – `GVBMR96` temporaries

`INITVAR` is carved out of the thread area during initialisation only. It
holds the cursors `GVBMR96` needs while converting the logic table:
`REGSAVE` (the mother's registers), `DRIVLRID`/`DRIVFILE`/`DRIVDDN` (current
driver LR and file), `FRSTRETK`/`PREVRETK`, `FIRSTRE`/`FIRSTNV`/`FIRSTKNV`,
`CURRRE`/`CURRNV`/`PREVNV`/`PREVLU`/`PREVWR`/`PREVES`,
`PREVDISK`/`PREVTAPE` (tails of the disk/tape queues), `ESCODBEG` (start of
the ES machine code) and `PREVLITP` (chapter 4).

## 12.14 Format-phase blocks

`WORKAREA` (`MAC/GVBMR88W.mac`) is the single R13 area of `GVBMR87`/`GVBMR88`.
Beyond the standard save areas it carries: `SAVESORT` (around the SORT
E15/E35 calls), `statflg1..4` (missing DDs, sort started/ended,
terminate-sort-on-error `x'ff'`), `srttime1..3` (SORT timing), `PARMDADR`,
`ENVVTBLA`, `SVRUN#`/`SVRUNDT`/`SVPROCDT`/`SVPROCTM`/`SVFINPDT`,
`SVRECCNT`, `SVVIEW#`, `SVFILENO`, `svpg#adr DS 40a` / `svpg#ftr DS 40a`
(deferred page-number patch addresses in headers/footers), `LBCHAIN` and
`VDP_MR88_View_list`.

The report-shape DSECTs `VIEWREC`, `SORTKEY`, `COLDEFN`, `CALCTBL`, `RTITLE`,
`HDRREC`, `CTLREC` are covered in chapter 10. The format-phase exit list
(`MAC/GVBAX88P.mac`) differs from the extract one:

```asm
GENPARM  DSECT               GENRPT  DSECT               GENRUN  DSECT
GPVWIDA  DS AL04 view id     GPRPTSEC DS CL02 section    GPRUN#   DS FL04
GPPRTLNA DS AL04 print line  GPCURLN# DS HL02 line no    GPRUNDT  DS 0CL08
GPSTARTA DS AL04 startup     GPCURPG# DS PL04 page no    GPPROCDT DS 0CL08
GPREPDTA DS AL04 -> GENRPT   GPMAXPG  DS HL02            GPPROCTM DS 0CL06
GPRUNDTA DS AL04 -> GENRUN   GPMAXLN  DS HL02
GPOUTPTA DS AL04 output rec  GPLINLEN DS HL02
GPWORKA  DS AL04 work anchor GPFINPDT DS 0CL06 fiscal period
                             GPCOMPNM DS CL80
                             GPTITLE  DS CL80
                             GPOWNER  DS CL08
```

## 12.15 Register conventions recap

| Register | Generated code / `GVBMR95` (chapter 6.3) | `GVBMR88` (chapter 10.2) |
|----------|-------------------------------------------|--------------------------|
| R2 | literal pool address minus 512K | work |
| R3, R4 | work; R3 = previous record for `..P` operands | work |
| R5 | current reference record / lookup buffer | loop counter / current reference record |
| R6 | current event record (`NVLEVNT LG R6,...`) | current `SORTKEY`, `COLDEFN` or `LKUPBUFR` |
| R7 | extract record (`EXTREC`) | current extract record (`EXTREC`/`HDRREC`/`CTLREC`) |
| R8 | current extract column (`NVLAYR8`) | current `VIEWREC` |
| R9, R10 | subroutine return (2nd / 1st level) | subroutine return |
| R11 | `NVCONST` of the current view | program base |
| R12 | `GVBMR95` base (V-con access) | – |
| R13 | `THRDAREA` | `WORKAREA` |
| R14, R15, R0, R1 | linkage / work (`BASSM`/`BSM` across AMODEs) | linkage / work |

Where a chapter documents a specific routine with different register usage
(for example the `GVBSRCH` search routines in chapter 11 or the `GVBMR96`
initialisation code in chapter 4), that chapter's table takes precedence.
