# 3. GVBMR95 — the extract engine

Source: `ASM/GVBMR95.asm` (~28,100 lines). Load module `GVBMR95`, aliases
`GVBMR95E` (extract) and `GVBMR95R` (reference phase). Contains the main task,
the sub-task (thread) entry `MR95THRD`, the generated-code *model* skeletons
(csect `MACHCODE`), the function table (`GVBFUNTB`/`GVBFUNCT`) and the
run-time support routines called from generated code (lookups, writes,
formatting, exits).

## 3.1 Life cycle at a glance

```
 EXEC PGM=GVBMR95E / GVBMR95R
   │
   ▼
 GVBMR95 (main task)                                    per event-file thread (MR95THRD)
 ┌────────────────────────────────────────────┐        ┌────────────────────────────────────────┐
 │ 1 which alias? (GVBURALI) ─ reject GVBMR95 │        │ Subtask: BAKR, SAM64, R13=THRDAREA      │
 │ 2 STORAGE OBTAIN main THRDAREA, zeroed     │        │ MAIN:   FP regs, allocate PETs (IEA4APE)│
 │ 3 EXITINIT (LE pre-init for COBOL exits)   │        │         EXITINIT, ESTAEX taskabnd       │
 │ 4 call GVBMR96  (init + code generation)   │        │ RESTART: reset work area for this ES set│
 │ 5 WLM enclave / zIIP init (if APF + USE_ZIIP)       │         run LTESINIT generated code     │
 │ 6 IDENTIFY EP=MR95THRD                     │        │         open event file via I/O driver  │
 │ 7 ATTACHX one thread per ES set (limits)   │──────► │ CALLEXIT: call write/lookup exits (OP)  │
 │ 8 EVENTS wait loop until THRDDONE=THRDCNT  │        │ header record handling (NOHDR/SKIPHDR..)│
 │   ESTAE slot → STATUS STOP/START + DETACH  │        │ EVNTLOOP: next record / EVNTBUFR refill │
 │ 9 close extract files, write control       │◄────── │ EVNTCODE: R6=record, BR LTESCODE        │
 │   report (EXTRRPT), timing (GVBURZTM)      │        │ EVNTEOF: close, WRTHDR, PIPEEOF, counts │
 │10 return code (0/8/12/16)                  │        │         next ES set? RESTART : CLPHASE  │
 └────────────────────────────────────────────┘        └────────────────────────────────────────┘
```

## 3.2 Alias enforcement

`GVBMR95` calls `GVBURALI` (`v(gvburali)`) which returns the alias the load
module was invoked with (or `GVBMR95` and RC 4 if none). The result is stored
in `NAMEPGM`. Executing the base name is an error:

```asm
if clc,namepgm,eq,=cl8'GVBMR95'
  GVBMSG WTO,MSGNO=EXEC_PGM_ERR,SUBNO=1, ...
  LHI   r15,12
  j     ERROREND
endif
```

The alias drives several later decisions: `GVBMR96` switches DDNAME prefixes
`EXTR`→`REFR` for the reference phase; `EVNTEOF` prints the
`THREAD_FINISHED` message only for `GVBMR95E` or when a DDNAME is present.

## 3.3 Main work-area allocation

```asm
STORAGE OBTAIN,LENGTH=THRDLEN+l'maineyeb,COND=NO,CHECKZERO=YES
```

The main task's `THRDAREA` (same DSECT as every thread's area) is obtained and
zeroed (the `CHECKZERO` result is tested and an `MVCL` clears the area if the
storage manager did not). The eyecatcher `MAINEYEB DC CL8'THRDMAIN'` precedes
it. R13 points at this area for the rest of the main task; every thread area
holds its address in `THRDMAIN`.

## 3.4 Initialization call

```asm
llgf  R15,GVBMR96
bassm R14,R15
```

`GVBMR96` (AMODE 31) receives the main `THRDAREA` and performs everything
described in [04-gvbmr96-initialization.md](04-gvbmr96-initialization.md):
parameters, VDP, logic table, lookups, thread areas, literal pools and code
generation. On return `THRDFRST` chains the thread work areas, `THRDCNT`
holds the number of threads to attach, and every `ES` row has `LTESCODE`
pointing at generated code.

Immediately after `GVBMR96` returns, `GVBMR95` itself runs **PASS2** of the
code generator (`LARL R15,PASS2` / `BASR R14,R15` at `gvbmr95s`), which
inserts operand offsets, adjusts literal-pool offsets and patches TRUE/FALSE
branch displacements into the copied skeletons — see
[06-generated-code.md](06-generated-code.md#65-pass2-gvbmr95). If
`EXECSNAP='Y'` (`DUMP_LT_AND_GENERATED_CODE`) the logic table and generated
code are then `SNAP`ped to `EXTRDUMP` (`REFRDUMP` under the `GVBMR95R`
alias).

## 3.5 Thread creation and synchronisation

The sub-task entry is made visible with `IDENTIFY EP=MR95THRD,ENTRY=(1)`
(failure → message `IDENTIFY_FAIL`, msg 001). Each thread is attached with

```asm
ATTACHX EP=MR95THRD,ECB=(7),SHSPV=15,SZERO=YES
```

with R7 → the thread's `TASKECB` and R1 → its `THRDAREA`. Disk, tape and
zIIP thread counts are bounded by `DISK_THREAD_LIMIT`, `TAPE_THREAD_LIMIT`
and `ZIIP_THREAD_LIMIT`; when the limit is hit, the remaining `ES` sets are
queued (`LTNXDISK`, `LTNXTAPE`, `LTNXOTHR` chains) and picked up by a
finishing thread through `PICKEVNT` (see 3.11). `EXECUTE_IN_PARENT_THREAD=Y`
(`EXECSNGL`) runs the first `ES` set on the main task instead of attaching.

Wait loop (AMODE 31 for `EVENTS`):

```asm
la    r4,1(,r2)              one slot per thread + 1 for the ESTAE ECB
sam31
EVENTS ENTRIES=(4)
sam64
...
waitloop do until=(clc,thrddone+2(l'thrdcnt),ge,thrdcnt)
         EVENTS Table=(4),wait=YES
         llgt r1,0(,r1)          posted ECB address
         if clrj,r1,eq,r7        ESTAE ECB posted?
           ... estae_stop, STATUS STOP, STATUS START,TCB=(2), DETACH (1),STAE=YES
         else
           ... DETACH (1),STAE=NO ; thrddone += 1
```

A thread whose ESTAE exit posted the shared "estae" ECB causes the main task
to stop all sub-tasks (`STATUS STOP`), restart the failing one so it can clean
up, and detach it with `STAE=YES`; normal completions are detached with
`STAE=NO`.

Detailed threading, PET (pause element), SRB/TCB and enclave behaviour is in
[08-threading-ziip-recovery.md](08-threading-ziip-recovery.md).

## 3.6 Thread start (`Subtask` → `MAIN` → `RESTART`)

```asm
Subtask  BAKR  R14,0        ; AR-mode program, linkage stack
         SAM64
         sac   0
         larl  r12,gvbmr95
         lae   r13,0(r1,0)   ; R1 → THRDAREA passed by ATTACHX
         llgtr r13,r13
         using (thrdarea,thrdend),r13
         ...
         mvc   4(4,r13),=c'F1SA'
MAIN     mvi   thread_mode,c'P'   ; problem state (TCB mode)
```

`MAIN` then:

1. Saves FP8–FP15 and loads the DFP quantum (`dfp_quantum_ptr`, set by
   MR96) into FP9/FP11 — used by decimal-floating-point model code.
2. Allocates TCB and SRB **pause elements** via `IEA4APE` (`IEAVAPE`), in
   supervisor state/key 0 if APF authorised (`localauth = 'A'`), else
   unauthorised PETs. The comment notes that the `WRTx` routines "have to
   single thread" and use pause/release.
3. Applies program patches (`CHKPATCH`/`BRNPATCH`), calls `EXITINIT`
   (LE pre-initialisation for COBOL exits, AMODE 31).
4. If `EXEC_ESTAE='Y'` sets the program mask for decimal/fixed overflow
   (`ovflmask`) and creates the recovery exit:
   `ESTAEX (r3),CT,PARAM=(R2),PURGE=HALT` with R3 → `taskabnd`.
5. Zeroes `thrd_evntrec_cnt`/`thrd_evntbyte_cnt`.

`RESTART` is entered once per `ES` set the thread processes:

* clears `LKUPKEY`, `GPDDNAME`, `GPLFID`, `EVNTSUBR`, `EVNTREAD`,
  `gp_call_srb`, `GP_redrive`, `eofevnt`, the `EXTREC` area;
* stores `THRDES` (current `ES` row) and `LTTHRDWK` (thread ↔ row link);
* loads the view literal pool: `R2 = LTESLPAD + 512K` — literal pools are
  addressed from their *middle* so that signed 20-bit displacements (`LAY`,
  `STY`, `LGF`…) reach ±512K (`using litp_hdr+524288,r2`);
* runs the generated `ES` **initialisation code** `LTESINIT` if present,
  temporarily redirecting the `NVNXVIEW` prolog pointer to `INITRETN`.

## 3.7 Opening the event file

If the `ES` set has a read row (`LTFRSTRE` → `RE` row) the thread copies
`LTDDNAME`, `LTFILEID`, `LTRENAME`, `LTREADDR`, `ltrePFcnt` into the
`GENFILE` area, patches the event DCB (`DCBDDNAM`, `DCBE` EODAD → `EVNTEOF`,
SYNAD → `SYNADEX0`), dynamically allocates the data set (`DYNALLOC` → SVC 99)
when `LTDSNAME` is given and the access method is not DB2/Adabas/MQ/pipe, and
dispatches on the access method (`LTACCMTH`, constants in `GVBMR95C.mac`):

```asm
select cij,r0,eq
  when SEQFILE   (01)  llgf R15,GVBMRBS
  when KSDSFILE  (03)  llgf R15,GVBMRVK
  when DB2SQL    (06)  llgf R15,MRSQADDR   ; WXTRN; 0 → DB2_SQL_UNAVAILABLE
  when DB2HPU    (16)  llgf R15,MRSUADDR   ; WXTRN; 0 → DB2_HPU_UNAVAILABLE
  when CALLADA   (17)  llgf R15,MRADADDR   ; WXTRN; 0 → ADABAS_UNAVAILABLE
  when DB2VSAM   (07)  llgf R15,MRDVADDR   ; WXTRN; 0 → DB2_VSAM_UNAVAILABLE
  othrwise             LHI r14,IO_DRIVER_UNAVAILABLE
endsel
LA    R0,LTREPARM ; read-exit parameters → GPSTARTA
bassm R14,R15     ; driver "initialize" call (GPPHASE = 'OP')
```

Each driver is called with the same `GENPARM` list and the phase code in
`GPPHASE` (`OP` open, then reads, `CL` close). The driver returns the address
of its *read* routine in `EVNTREAD` and fills `GENFILE` (`GPRECFMT`,
`GPRECLEN`, buffer addresses, `EODADDR`). For sequential files `EVNTREAD`
is inside `GVBMRBS` and, per the "DANGER Will Robinson" comment, assumes R2 →
DCB and clobbers registers.

### Exit initialisation (`CALLEXIT`)

The thread walks the `NV`…`WR` rows of the set: every `WR` row with an exit
name (`LTWRNAME`) has its write exit called with `GPPHASE='OP'`, its
work-area anchor `LTWRWORK` and parameter string `LTWRPARM`. Return codes
> 8 disable the view (12) or abort (`abortex`); exits may also supply
`gp_error_reason`/`gp_error_buffer_ptr` text which is written to the log
under an `ENQ (GENEVA,LOGNAME)`. Lookup exits (`LK` rows with an exit) are
initialised the same way (`LKPXLEIN` stub).

### Header records

`LTHDROPT`/`GPHDROPT` selects `NOHDR` (01), `VERHDR` (02, verify the header:
`LTVERNO` must match, else msg 006 "Source file header record version
wrong") or `SKIPHDR` (03). Header records of extract files produced by a
previous pass are recognised through the `EXTREC` layout.

## 3.8 The record loop

```asm
EVNTLOOP llgt  R2,EVNTDCBA
         agf   R6,GPRECLEN          ; advance to next record
         cg    R6,EODADDR           ; end of buffer?
         JNL   EVNTBUFR             ;   yes → refill
         TM    DCBRECFM,X'80'       ; fixed/undefined?
         JO    evntcode
         LLH   R0,0(,R6)            ; variable: RDW length
         AHI   R0,-4
         ST    R0,GPRECLEN
         AHI   R6,4                 ; step over RDW
         J     evntcode

EVNTBUFR ; zIIP: if in SRB mode and driver is not SRB-savvy → TCB_switch
         do ,
           sgr   R7,R7
           llgf  R15,EVNTREAD
           BASSM R14,R15            ; driver fills next buffer
           if workazip≠0 and GP_redrive='Y'  → TCB_switch ; iterate (redrive I/O)
         enddo
         ; zIIP: if in TCB mode → SRB_switch (run generated code on zIIP)

EVNTCODE STG   R6,RECADDR
         LAY   R0,RECADDR
         STY   R0,GPEVENTA
         cgijnh r6,0,evntnorec      ; no record (reference phase)
         agsi  GPRECCNT,bin1
evntnorec llgt R7,GPEXTRA           ; R7 → EXTREC
         llgt  R8,THRDES            ; R8 → ES row
         LLGT  R2,THRDLITP
         agfi  r2,f512k             ; R2 → literal pool middle
         llgt  R15,LTESCODE
         BR    R15                  ; → generated code
```

Generated code processes the record against every view of the set and ends
by branching back to `EVNTLOOP` (the address is planted in the generated
epilogue by MR96). When the reader hits end of file the DCBE EODAD exit
`EVNTEOF` is driven (or the driver returns EOF and branches there).

The generated code, for each view (`NV` prolog), evaluates selection logic
(`CF*`, `CN*`, …), performs lookups (`LK*`/`LU*`/`JOIN`), builds columns
(`DT*`, `CT*`, `SK*`, `KS*`), accumulates (`ADD*`, `SUB*`, `SET*`, …) and
issues `WR*` — a call back into MR95's write routines (`wrtxbypd`,
`wrtdbypd`, `wrttbypd`, `wrtsbypd` for extract/DT-area/token/summary
writes). Details in [06-generated-code.md](06-generated-code.md).

## 3.9 End of event file (`EVNTEOF`)

```
EVNTEOF  sam64                                    (entered in AMODE 31 from DCBE)
         if SRB mode → TCB_switch
         clear gpview#
         THREAD_FINISHED message (thread no., record count)   [GVBMR95E only]
         PGSER R,FREE   if buffers were page-fixed
         CLOSE ((R3),FREE),MODE=31 ; Free_Evnt_io_buf
CKPIPE   WRTHDR   write header records to extract files of this ES set
         PIPEEOF  bump EOF counters for pipe-creating threads (wake readers)
         LTFILCNT ← GPRECCNT ; thread totals thrd_evntrec_cnt / thrd_evntbyte_cnt
         if LTSAMES# (concatenated file in same thread)  → RESTART
         if not EXECSNGL: PICKEVNT → next compatible ES set → RESTART
         CLPHASE  call exits/drivers with GPPHASE='CL'
         retncode ← 0 ; SRB_end (zIIP) ; ESTAEX 0 ; SPM 0 ; PR (return to ATTACH)
```

## 3.10 Extract-record writes and output files

* `EXTREC` (`GVBMR95C.mac`): `EXRECLEN` (RDW), `EXSORTLN`, `EXTITLLN`,
  `EXDATALN`, `EXNCOL`, `EXVIEW#`, then sort key, sort title, data. Max
  32756.
* Extract files are described by `EXTFILE` entries built by MR96 from the
  `WR` rows (`LTWRFILE`, `LTWREXTO` → `LTWRAREA` in the literal pool with
  `LTWRWORK`, counters). Several views/threads may write to the same DDNAME;
  the write routines serialise with the PET pause/release protocol
  (`XFRTCB*`, `TCBPET1/2`).
* Output is BSAM `WRITE` of blocks; `GENWRITE` is used where an exit
  requests it (`GVBUR39`).
* Device types (`LTFILTYP`): `DISKDEV` 02, `TAPEDEV` 03, `PIPEDEV` 04,
  `TOKENDEV` 05. Pipes are memory buffers (`PIPEBUFR EQU 3`) between a
  writing thread and a reading `ES` set in another thread; tokens
  (`LTRTOKEN`, `WR_TK`, `ET` rows) hand a record to a consuming view in the
  *same* thread.
* `WRTHDR` writes the extract-file header record (control data used by
  `GVBMR88` `PROCESS_HEADER_RECORDS`).

## 3.11 Work-unit selection (`PICKEVNT`)

`PICKEVNT` (R1 → thread area, R10 return) selects the next pending `ES` set
from three queue heads in the main `THRDAREA`, chosen by the thread's type
(`THRDTYP`):

```
 THRDTYP = TAPEDEV : LTNXTAPE, else LTNXDISK
 THRDTYP = DISKDEV : LTNXDISK only
 other             : LTNXOTHR (DB2, pipes, tokens, Adabas…), else LTNXDISK
```

The queue is a lock-free singly linked list: the head is dequeued with
`CS R8,R0,0(R15)` (compare-and-swap of `LTNEXTES`), retrying from
`PICKEVNT` on contention. The chosen row is appended to the thread's
`THRDEXEC` list and returned in R8; `EVNTEOF` then jumps to `RESTART`.

## 3.12 Error handling and return codes

* `ERRMSG#` — common message exit: R14 = message number, R15 = return code;
  writes via `GVBMSG` and continues to `THRDMSG`/`ERROREND`.
* `taskabnd` — ESTAE exit for threads: records the abend, takes a `SNAP` to
  `EXTRDUMP` if `RECOVER_FROM_ABEND` allows, posts the ESTAE ECB so the main
  task can stop the other threads. `ABEND_ON_*` parameters convert selected
  conditions into user abends (`GVBUT99`-style `ABEND ...,DUMP`).
* `PE_error`/`PE_FAIL` — pause element service failure → log + RC 16.
* Return codes seen in source: **0** success; **4** warnings (e.g. `GVBURALI`
  no alias); **8** processing/parameter error (`ERRMSG#` default);
  **12** `EXEC_PGM_ERR` (direct execution of `GVBMR95`), exit "disable view";
  **16** fatal (PE failure, abend recovery, `abortex`).

## 3.13 Control report

At end of job the main task writes `EXTRRPT`: parameter echo (from MR96),
per-thread and per-file record/byte counts (`ltfilcnt`, `thrd_evntrec_cnt`),
per-view extract counts (`LPEXTCNT`), lookup found/not-found counts
(`LPLKPFND`/`LPLKPNOT`, `lbfndcnt`/`lbnotcnt`), write-unit statistics (the
`twrtrept` temporary DSECT sorts output DDNAMEs), and the CPU/zIIP/enclave
time report produced by `GVBURZTM` (`IWMEQTME`).
