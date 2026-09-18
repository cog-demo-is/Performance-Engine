# 08 - Threading, zIIP Offload and Recovery

`GVBMR95` processes every event-file/"ES" set on its own z/OS subtask and,
optionally, runs the generated machine code under an SRB so that it is
eligible for zIIP specialty engines. This chapter documents the task model,
the synchronization primitives (EVENTS, compare-and-swap queues, pause
elements), the zIIP/WLM enclave integration, and the ESTAE recovery path.
Source: `ASM/GVBMR95.asm`, `MAC/GVBMR95W.mac`, `MAC/GVBMRZPE.mac`,
`MAC/EXECDATA.mac`.

## 1. Task model

```text
   mother task (GVBMR95 main TCB)
   +--------------------------------------------------------------+
   | STORAGE OBTAIN thread area (THRDLEN)                          |
   | CALL GVBMR96 (init, builds THRDAREA chain and generated code) |
   | IDENTIFY EP=MR95THRD,ENTRY=Subtask                            |
   | EVENTS ENTRIES=THRDCNT+1                                      |
   | ATTACHX ... x THRDCNT   -----+-----------+-----------+        |
   | waitloop: EVENTS WAIT=YES    |           |           |        |
   +------------------------------|-----------|-----------|--------+
                                  v           v           v
                            daughter 1   daughter 2   daughter n   (TCBs)
                            R13 -> own THRDAREA (from GVBMR96 THRDBLD)
                            takes ES work units from LTNXDISK/TAPE/OTHR
                            optional: schedules an SRB (GVBMRZP) that
                            runs the generated code on a zIIP
```

### 1.1 Thread work area

Each thread owns a `THRDAREA` (`MAC/GVBMR95W.mac`), allocated by `GVBMR96
THRDBLD` and chained through `THRDNEXT`. The mother's area is `THRDMAIN`;
the first daughter is `THRDFRST`. Fields used by the scheduling code:

| Field | Purpose |
|-------|---------|
| `THRDMAIN` | address of the mother thread area (shared counters live here) |
| `TCBADDR` | TCB address returned by `ATTACHX`; zeroed after `DETACH` |
| `TASKECB` | ECB posted when the subtask ends (or by the ESTAE exit) |
| `estae_stop` | non-zero while `STATUS STOP` has been issued by the mother |
| `THRDDONE` | count of daughters that have finished |
| `THRDCNT` | number of daughters attached |
| `Thread_fail` | id of the first failing thread (updated with `CS`) |
| `Thread_mode` | `C'P'` problem state (TCB) or `C'S'` supervisor state (SRB) |
| `CODEBEG` / `CODEEND` | bounds of this thread's generated-code buffer, used by the ESTAE exit to map an abend PSW back to a logic-table row |
| `WAITECB`, `EXTINUSE` | extract-file serialization (section 3) |
| `XFRTCBPL` / `XFRSRBPL` | `IEA4APE/IEA4PSE/IEA4RLS` parameter lists for the TCB and SRB pause elements |
| `tcbpet1a` / `srbpet1a` | pause element tokens |
| `workazip` | address of `GVBMRZP` with the AMODE-64 bit set, or zero |

### 1.2 Entry point and IDENTIFY

`MR95THRD` is created at run time with `IDENTIFY EP=MR95THRD,ENTRY=(1)`
(R1 = address of `Subtask`). A comment block in the source explains why the
subtask starts with `SAM64` immediately: `IDENTIFY` inherits the AMODE of the
module (31), so the subtask switches itself to 64-bit, clears the high halves
of all registers, zeroes AR0-AR10/AR14-AR15 and writes `F1SA` into the save
area back-chain to signal that the linkage stack (`BAKR`) is in use:

```asm
Subtask  BAKR  R14,0              stack the gprs,ars mode etc.
         SAM64
         sac   0                  make sure we are in primary mode
         larl  r12,gvbmr95        set up 1st base
         lae   r13,0(r1,0)        copy the address and alet
         llgtr r13,r13            and clean it
         ...
         XC    LKUPKEY,LKUPKEY    ZERO   ACCESS   REGISTER AREA
         LAM   R0,R10,lkupkey     ZERO ar0-ar10
         LAM   R14,r15,lkupkey    and ar14-ar15
         mvc   4(4,r13),=c'F1SA'
```

### 1.3 Attach and the EVENTS wait loop

The mother allocates an `EVENTS` table with `THRDCNT+1` slots: one per
daughter ECB plus one for the ESTAE ECB (`mother.taskecb`). Each daughter is
attached with `ATTACHX EP=MR95THRD,ECB=(7),SHSPV=15,SZERO=YES`, and its
`TASKECB` address is added to the table. The wait loop:

```asm
waitloop do until=(clc,thrddone+2(l'thrdcnt),ge,thrdcnt)
           EVENTS Table=(4),wait=YES Now wait for a subtask to finish
           llgt r1,0(,r1)         get ecb address
           if clrj,r1,eq,r7       Is it the one set up for the ESTAE?
             st r7,estae_stop
             llgt r2,0(,r1)       ECB contents = failing TCB address
             nilf r2,x'00ffffff'
             status STOP          stop all the daughters
             status START,TCB=(2) and allow the failing task to finish
           else
             asi thrddone,1
             llgt r15,0(0,r1)     completion code
             nilf r15,x'00ffffff'
             if cl,r15,gt,overall_return_code
               st r15,overall_return_code
             endif
             if chi,r15,gt,900,or,lt,r0,estae_stop,nz
               ... DETACH (1),STAE=YES every thread with TCBADDR != 0
               if estae_stop: STATUS START
               leave ,
             else
               DETACH (1),STAE=NO   normal end of one daughter
             endif
           endif
         enddo
         events entries=DEL,table=(4)
```

Decision table for a posted ECB:

| Posted ECB | Completion code | Action |
|------------|-----------------|--------|
| ESTAE ECB (`mother.taskecb`) | TCB address of the abending thread | `STATUS STOP` all daughters, `STATUS START,TCB=` the failing one so its ESTAE exit can finish |
| daughter `TASKECB` | `<= 900` and no ESTAE active | `DETACH ,STAE=NO`, keep waiting |
| daughter `TASKECB` | `> 900` (user abend) or `estae_stop` set | `DETACH ,STAE=YES` on every remaining thread (drives their ESTAE exits so outstanding SRBs are cleaned up), `STATUS START`, leave loop |

`overall_return_code` becomes the job step return code after the normal
end-of-run processing.

### 1.4 Single-thread modes

`EXECSNGL` (`EXECUTE_IN_PARENT_THREAD` in `MR95PARM`, values `1`, `A` or
`N`) changes the model:

| `EXECSNGL` | Meaning |
|------------|---------|
| `C'1'` | run only the first ES set, on the mother TCB (debugging aid) |
| `C'A'` | all ES sets processed sequentially on the mother TCB, no `ATTACHX`, no ESTAE ECB post |
| `C'N'` (default) | full parallel mode; `EXECDISK`/`EXECTAPE` (`DISK_THREAD_LIMIT`/`TAPE_THREAD_LIMIT`) cap the disk and tape threads |

The ESTAE exit checks the same flag before posting the mother.

## 2. Work-unit selection (`PICKEVNT`)

The ES logic-table rows are pre-sorted by `GVBMR96` into three lock-free
singly linked lists anchored in the mother area: `LTNXDISK`, `LTNXTAPE` and
`LTNXOTHR` (non-sequential sources such as DB2, Adabas, VSAM, pipes). A
thread's `THRDTYP` decides which list it drains first; an empty list falls
back as shown:

```text
   THRDTYP = TAPEDEV  ->  LTNXTAPE, then LTNXDISK
   THRDTYP = DISKDEV  ->  LTNXDISK
   otherwise          ->  LTNXOTHR, then LTNXDISK
```

Removal from a list is a compare-and-swap, so no ENQ/lock is taken; a
failed `CS` restarts the whole selection:

```asm
PICKEVNT llgt  R14,THRDMAIN         LOAD MAIN THREAD WORK AREA ADDRESS
         LH    R0,THRDTYP-THRDAREA(,R1)  LOAD THREAD TYPE
PICKTAPE LA    R15,LTNXTAPE-THRDAREA(,R14)    ASSUME TAPE
         CHI   R0,TAPEDEV                     TAPE THREAD ???
         JNE   PICKDISK
         ltgf  R8,0(,R15)                     ANY TAPES LEFT ???
         JP    PICKSWAP
         LA    R15,LTNXDISK-THRDAREA(,R14)    ANY DISKS LEFT ???
         ...
PICKSWAP llgt  R0,LTNEXTES                    UPDATE NEXT "ES" POINTER
         CS    R8,R0,0(R15)
         JNE   PICKEVNT
         XC    LTNEXTES,LTNEXTES              RESET  NEXT "ES" POINTER
         ltgf  R14,THRDEXEC-THRDAREA(,R1)     ANY FILES ALREADY PICKED
         JP    PICKEXEC
         ST    R8,THRDEXEC-THRDAREA(,R1)
         J     PICKEXIT
PICKEXEC lgr   R15,R14                        append to this thread's
         ltgf  R14,LTNEXTES-LOGICTBL(,R14)    own chain (THRDEXEC)
         JP    PICKEXEC
         ST    R8,LTNEXTES-LOGICTBL(,R15)
PICKEXIT BR    R10
```

The removed ES row is appended to the thread's private `THRDEXEC` chain, so
the control report can list which thread processed which event file.
Threads load-balance dynamically: a thread that finishes a small event file
immediately takes the next ES set.

## 3. Extract-file serialization with pause elements

Several threads may write to the same extract file (`EXTRnnn`). The file's
control area holds `EXTINUSE`; a writer claims it with `CS` and, if it is
already held, links its own `WAITECB` onto a waiter chain and *pauses* on a
z/OS pause element instead of `WAIT`ing on an ECB (pause elements work in
both TCB and SRB mode):

```asm
WRTXENQ  LA    R1,WAITECB
         XR    R14,R14
         ST    R14,0(,R1)
         LNR   R0,R1              NEGATIVE VERSION OF ECB ADDRESS
WRTXRTRY CS    R14,R0,EXTINUSE    TEST IN-USE FLAG ???
         JE    WRTXNEW            BRANCH  IF  ZERO (NOT IN-USE)
         ST    R14,4(,R1)         SAVE PREVIOUS VALUE IN LINKED LIST
         CS    R14,R1,EXTINUSE    MOVE THIS THREAD'S VALUE TO FLAG
         JNE   WRTXRTRY
         if    cli,thread_mode,eq,c'S'
           ... xfrsrbpl / srbpet1a, IPK + SPKA 0 (key 0)
         else
           ... xfrtcbpl / tcbpet1a, MODESET KEY=ZERO,MODE=SUP if localauth=A
         endif
         llgt R15,IEA4PSE
         BASR R14,R15             pause until released
```

When the owner finishes its write it pops the first waiter, looks at that
waiter's `thread_mode` to choose the SRB or TCB pause element token, and
calls `IEA4RLS`. The `EXTINUSE` word therefore has three states:

```text
   0                 free
   negative value    held, no waiters (-(WAITECB) of the owner)
   positive address  held, points at a chain of waiting WAITECBs
```

Pause elements are allocated once per thread at start-up with `IEA4APE`. With
APF authorization (`localauth = C'A'`, tested with `TESTAUTH FCTN=1`) an
authorized PE (`PETAUTH`) is allocated under `MODESET KEY=ZERO,MODE=SUP`;
otherwise `PET_notauth` is used. Failure of any PE service ends the thread
through `PE_error`, which logs the service name and return code.

The `V(IEA4APE)`, `V(IEA4PSE)`, `V(IEA4XFR)`, `V(IEA4RLS)` and `V(IEA4DPE)`
address constants are resolved at link time from the z/OS system linkage
library (`CSSLIB`).

## 4. zIIP offload

### 4.1 Contract with `GVBMRZP`

The zIIP support lives in a separate module, `GVBMRZP`, referenced as a weak
external (`WXTRN GVBMRZP` / `ZIIPADDR DC V(GVBMRZP)`). It is **not part of this
repository**; only its function-code interface (`MAC/GVBMRZPE.mac`) and the
call sites are. If the module is absent and `ZIIP=Y` was requested, `GVBMR95`
logs `ZIIP_FEATURE_NOT_AVAILABLE` and ends with return code 8.

```asm
zIIP_init  EQU  1     ZIIP initialisation
zIIP_oct   EQU  2     offload control (after enclave join)
SRB_sched  EQU  3     Schedule SRB and process record
TCB_switch EQU  4     Switch to TCB mode
SRB_switch EQU  5     Switch to SRB mode
SRB_end    EQU  6     Clean up the SRB
```

Every call has the same shape:

```asm
         if (ltgf,r15,workazip,nz)     zIIP function available?
           la    r1,<function code>
           bassm r14,r15               Call zIIP module (AMODE 64)
         endif
```

`workazip` is set once in the mother from `ZIIPADDR` with bit 63 turned on
(`oill r15,x'0001'`) so `BASSM` enters the module in AMODE 64, and copied into
each daughter's area during thread build.

### 4.2 Mode switches during event processing

`Thread_mode` tracks where the thread is executing. The generated code and
lookups are pure CPU work and run in SRB mode; I/O macros and most exits must
run under the TCB. The switches are placed around the buffer refill loop:

```asm
ProcRec  if (ltgf,r15,workazip,nz)
           la    r1,SRB_sched        Schedule SRB and process record
           bassm r14,r15
         else
           BR    R10                 Go process record (call Machcode)
         endif

EVNTBUFR if (ltgf,r15,workazip,nz),and,(cli,thread_mode,eq,C'S')
           if (ltgf,r14,thrdre,p),and,(tm,ltflag2,ltnomode,z),and,   +
               (cli,gp_call_srb,ne,c'Y')
             la  r1,TCB_switch       Switch to TCB mode
             bassm r14,r15
           endif
         endif
         do ,
           llgf  R15,EVNTREAD
           BASSM R14,R15             FILL  NEXT BUFFER
           if (ltgf,r15,workazip,nz),and,(cli,GP_redrive,eq,c'Y')
             la  r1,TCB_switch       routine asked for a switch
             bassm r14,r15
             iterate ,               and redrive the i/o
           endif
         enddo
         if (ltgf,r15,workazip,nz),and,(cli,thread_mode,eq,C'P')
           la    r1,SRB_switch       Switch to SRB mode
           bassm r14,r15
         endif
```

```text
   SRB (zIIP eligible)                  TCB
   +-----------------------+            +--------------------------+
   | generated code        |  buffer    | EVNTREAD (BSAM/VSAM/DB2) |
   | lookups, extract      |  empty     | exits that need TCB      |
   | record build          | ---------> | write extract blocks     |
   |                       | <--------- |                          |
   +-----------------------+  SRB_switch+--------------------------+
```

Read routines that are "SRB savvy" set `gp_call_srb = C'Y'` in the
`GVBX95PA` area and are called without leaving SRB mode; a routine that finds
itself in SRB mode but needs the TCB sets `GP_redrive = C'Y'` and returns, and
`GVBMR95` switches and calls it again. None of the drivers shipped in this
repository (`GVBMRBS`, `GVBMRVK`, `GVBMRSQ`, `GVBMRSU`, `GVBMRAD`) set either
flag, so in practice every read is preceded by `TCB_switch`; the hooks exist
for external drivers. The `LTNOMODE` logic-table flag
suppresses switching for an ES set entirely. Trace output (`TRACSUBR`) and
user exits also force `TCB_switch` before calling z/OS services.

`EXEC_SRBLIMIT` (`ZIIP_THREAD_LIMIT` parameter) caps the number of
concurrently active SRBs, and `execovfl_on`/`ovflmask`
(`ABEND_ON_CALCULATION_OVERFLOW`) sets PSW bits 36-37 (fixed-point and
decimal overflow masks) via `SPM` at thread start and after every pause.

### 4.3 WLM enclave

When zIIP is enabled the mother creates a dependent enclave and every
daughter joins it, so the SRB time is attributed to the job and reported:

```asm
         IWM4ECRE TYPE=DEPENDENT,ETOKEN=ENCLTOKN
         ...
         if authorized
           sysevent ENCASSOC,ENTRY=BRANCH,TYPE=ENCASSOC_JOIN
         else
           IWMEJOIN ETOKEN=ENCLTOKN
         endif
         la r1,zIIP_oct           zIIP offload control function
         bassm r14,r15
```

At end of run `IWMEQTME CPUTIME=ENC_CPUTIME,...` collects enclave CPU and
zIIP time for the control report (formatted by `GVBURZTM`), then the tasks
leave (`sysevent ENCASSOC,TYPE=ENCASSOC_LEAVE` or `IWMELEAV`) and the mother
deletes the enclave with `IWM4EDEL`. `encassoc_parm` is a 24-byte SYSEVENT
parameter list with function code 1 (join) or 2 (leave).

## 5. Recovery (ESTAE)

### 5.1 Establishing the exit

Unless `RECOVER_FROM_ABEND=N` (`exec_estae`, default `Y`) each subtask
establishes an ESTAEX exit pointing at `TASKABND`, passing its own thread
area:

```asm
         larl r3,taskabnd
         XC    WKREENT,WKREENT
         ESTAEX (r3),CT,PARAM=(R2),PURGE=HALT,MF=(E,WKREENT)
```

`PURGE=HALT` halts outstanding I/O so BSAM buffers are not written after the
failure. The exit is removed with `estaex 0` at normal thread end.

### 5.2 `TASKABND` flow

```text
   abend in daughter
        |
        v
   TASKABND (R0=12 -> no SDWA: just CS Thread_fail and percolate)
        |
        +-- STMG registers; R8 = thread area from SDWAPARM
        +-- POST mother.taskecb with own TCB address  --> mother STATUS STOPs
        |   (only in parallel mode)                        the other daughters
        +-- if Thread_mode = 'P' and zIIP active: SRB_end (clean up SRB)
        +-- copy PSW address (sdwanxt1), 64-bit registers (sdwag64),
        |   completion code (sdwacmpf) and reason code (sdwacrc)
        +-- CS Thread_fail := gpthrdno   (first failing thread wins)
        |     |
        |     +-- unless completion code is S33E: GVBMSG THREAD_ABEND
        |     |   (thread id, completion code, view id)
        |     +-- scan_lt: map failing PSW / R10 / R14 / R15 into the
        |     |   generated-code buffer [CODEBEG,CODEEND) and back to
        |     |   the logic-table row whose LTCODSEG contains it
        |     +-- GVBMSG FAILING_ROW (row number)
        v
   SETRP / return to RTM (percolate); mother DETACHes with STAE=YES
```

`scan_lt` walks the `NV` rows of the current ES set comparing the failing
address with each row's `LTCODSEG` range and returns `LTROWNO`, which is the
row number printed in the `FAILING_ROW` message - the primary debugging aid
for a failure inside generated code:

```asm
scan_lt  llgt  r14,thrdes         get the es logic entry
         llgt  r15,eslt.ltfrstnv  get the first nv pointer
lt_outer do ,
           do until=(clgrj,r15,ge,r14)
             lgr r1,r15
             agh r15,ltrowlen
             if clgf,r6,ge,current_lt.ltcodseg,and,clgf,r6,le,ltcodseg
               lgf r6,current_Lt.ltrowno
               leave lt_outer
             endif
           enddo
           xgr r6,r6               no entry found, so return 0
         enddo
         br    r9
```

Because `GVBMSG` (`errformat`) expects R13 to address the thread area, the
exit temporarily swaps R13 (`lgr r10,r13 / lgr r13,r8`) around each message
call and saves the live registers in `estae_rsa`.

### 5.3 User abends and return codes

* Thread completion codes above `U900` are treated as fatal for the whole run
  (see the wait loop). `GVBUT99` issues user abends when `exec_uabend = C'Y'`;
  otherwise the same condition becomes a non-zero return code.
* `abend_msg`/`abend_lt` (`ABEND_ON_MESSAGE_NBR`, `ABEND_ON_LOGIC_TABLE_ROW_NBR`)
  allow forcing an abend (`DC XL4'FFFFFFFF'`, S0C1) when a specific message
  number is issued or a traced logic-table row is executed, producing a dump
  with the failing row information above. `ABEND_ON_ERROR_CONDITION`
  (`exec_uabend`) turns error return codes into user abends.
  `INCLUDE_REF_TABLES_IN_SYSTEM_DUMP` (`EXEC_Dump_Ref`) is parsed and echoed
  but not read by any other statement in this source tree.
* `DUMP_LT_AND_GENERATED_CODE=Y` (`EXECSNAP`) makes the mother `SNAP` the
  logic table (ID 020) and each thread's generated code (ID 030) to the
  `SNAPDCB` file before processing starts - the usual way to inspect the code
  that PASS1/PASS2 produced.
* `overall_return_code` is the maximum of the daughters' completion codes and
  is returned in R15 after end-of-run reporting.

## 6. Timing statistics

The control report's CPU/zIIP/elapsed figures come from:

* `TIME STCK` at start and end of run (`begtime`, converted with `STCKCONV`)
  for elapsed time,
* `IWMEQTME CPUTIME=ENC_CPUTIME,...` for the enclave (CP time, zIIP time,
  zIIP-on-CP time) when zIIP is active,
* `GVBURZTM`, which formats the STCK differences and enclave values into the
  `EXTRRPT` summary lines.

See [03-gvbmr95-extract-engine.md](03-gvbmr95-extract-engine.md) for where
these values are printed and [13-parameters-ddnames-messages.md](13-parameters-ddnames-messages.md)
for the `USE_ZIIP`, `ZIIP_THREAD_LIMIT`, `RECOVER_FROM_ABEND`,
`DISK_THREAD_LIMIT`, `TAPE_THREAD_LIMIT` and `EXECUTE_IN_PARENT_THREAD`
parameters.
