# Oracle 21c DBCA Failure — Troubleshooting & Resolution Report

**Environment**: Oracle Linux 7.9 (Hyper-V), Kernel UEK 5.4.17  
**Software**: Oracle Grid Infrastructure 21c + Oracle Database 21c  
**Storage**: ASM on loop device (`/dev/loop1` → `/opt/asm-disks/asm_disk1.img`, 5 GB)  
**Date**: 2026-05-01  

---

## Step 1 — Run the DBCA Command

The following command was executed as the `oracle` user to create the Oracle 21c database:

```bash
su - oracle
export ORACLE_HOME=/u01/app/oracle/product/21c/dbhome
export ORACLE_SID=orcl
export PATH=$ORACLE_HOME/bin:$PATH

dbca -silent \
  -createDatabase \
  -templateName General_Purpose.dbc \
  -gdbName orcl \
  -sid orcl \
  -characterSet AL32UTF8 \
  -sysPassword Oracle123 \
  -systemPassword Oracle123 \
  -storageType ASM \
  -datafileDestination '+DATA' \
  -recoveryAreaDestination NONE \
  -databaseType MULTIPURPOSE \
  -totalMemory 800 \
  -emConfiguration NONE \
  -ignorePreReqs
```

**Observed output:**

```
Prepare for db operation
10% complete
Registering database with Oracle Restart
14% complete
Copying database files
43% complete
100% complete
[FATAL] Recovery Manager failed to restore datafiles. Refer logs for details.
14% complete
10% complete
0% complete
Look at the log file "/u01/app/oracle/cfgtoollogs/dbca/orcl/orcl10.log" for further details.
```

Progress jumped to 100% and then immediately rolled back to 0%, indicating DBCA detected a fatal error and aborted the database creation.

---

## Step 2 — Read the DBCA Log

```bash
cat /u01/app/oracle/cfgtoollogs/dbca/orcl/orcl10.log
```

```
[ 2026-05-01 20:17:15 EDT ] Prepare for db operation
DBCA_PROGRESS : 10%
[ 2026-05-01 20:17:19 EDT ] Registering database with Oracle Restart
DBCA_PROGRESS : 14%
[ 2026-05-01 20:17:20 EDT ] Copying database files
DBCA_PROGRESS : 43%
DBCA_PROGRESS : 100%
[ 2026-05-01 20:17:29 EDT ] [FATAL] Recovery Manager failed to restore datafiles.
DBCA_PROGRESS : 14%
DBCA_PROGRESS : 10%
DBCA_PROGRESS : 0%
```

The log confirms failure but gives no detail. The referenced log directory contains more files — check them next.

---

## Step 3 — Read the RMAN Restore Log

```bash
ls -lt /u01/app/oracle/cfgtoollogs/dbca/orcl/
cat /u01/app/oracle/cfgtoollogs/dbca/orcl/CloneRmanRestore.log
```

The log showed the Oracle instance started and mounted the database successfully. However, there was one earlier entry showing:

```
ORA-00838: Specified value of MEMORY_TARGET is too small, needs to be at least 492M
ORA-01078: failure in processing system parameters
```

This was from a previous DBCA run. The current run (orcl10) showed the instance started with 838 MB SGA and mounted — but then RMAN failed when trying to write files.

---

## Step 4 — Read the DBCA Trace Log

```bash
grep -i "ORA-\|error\|fatal\|rman" \
  /u01/app/oracle/cfgtoollogs/dbca/orcl/trace.log_2026-05-01_08-16-56PM \
  | tail -60
```

Key errors found:

```
channel ORA_DISK_1: ORA-19870: error while restoring backup piece
  /u01/app/oracle/product/21c/dbhome/assistants/dbca/templates/Seed_Database.dfb
ORA-19504: failed to create file "+DATA"
ORA-17502: ksfdcre:4 Failed to create file +DATA
ORA-15001: diskgroup "DATA" does not exist or is not mounted
ORA-01034: ORACLE not available
ORA-27123: unable to attach to shared memory segment
RMAN-03002: failure of restore command
ORA-01180: can not create datafile 1
```

The Oracle error stack reads **bottom-up** (the bottom error is the root cause):

| Order | Error | Meaning |
|---|---|---|
| Root | `ORA-27123` + `Linux Error: 13 Permission denied` | Cannot attach to ASM shared memory |
| ↑ | `ORA-01034: ORACLE not available` | ASM instance unreachable |
| ↑ | `ORA-15001: diskgroup "DATA" does not exist` | Cannot access +DATA |
| ↑ | `ORA-19504: failed to create file "+DATA"` | File creation failed |
| ↑ | `ORA-19870: error while restoring backup piece` | RMAN restore failed |

The root cause is `ORA-27123`: the ASMB (ASM Bridge) background process inside the Oracle DB instance cannot attach to ASM's shared memory segment.

---

## Step 5 — Read the Oracle Instance Alert Log

```bash
tail -80 /u01/app/oracle/diag/rdbms/orcl/orcl/trace/alert_orcl.log
```

```
ALTER DATABASE   MOUNT
Set as converted control file due to db_unique_name mismatch
WARNING: failed to start ASMB (connection failed) state=0x1 sid='+ASM'
Errors in file .../trace/orcl_asmb_2895.trc:
ORA-01034: ORACLE not available
ORA-27123: unable to attach to shared memory segment
Linux-x86_64 Error: 13: Permission denied
Additional information: 3508       ← shmid
Additional information: 1
Additional information: 1623195648
Stopping background process RBAL
Successful mount of redo thread 1, with mount id 265787925
Database mounted in Exclusive Mode
WARNING: ASMB exiting with error
...
WARNING: failed to start ASMB (connection failed) state=0x1 sid='+ASM'
WARNING: ASMB exiting with error
...
Shutting down ORACLE instance (abort)
Instance terminated by USER, pid = 3336
```

**Key findings:**

- The database **did mount** (it could talk to ASM initially via socket)
- But ASMB — the background process needed for ASM file I/O — failed 3 times with `Linux-x86_64 Error: 13: Permission denied`
- The shmid being accessed was **3508**
- Without ASMB, no files can be written to ASM, so RMAN cannot restore the seed database

---

## Step 6 — Read the ASMB Trace File

```bash
cat /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_asmb_2895.trc
```

```
WARNING: failed to start ASMB (connection failed) state=0x1 sid='+ASM'
ORA-01034: ORACLE not available
ORA-27123: unable to attach to shared memory segment
Linux-x86_64 Error: 13: Permission denied
Additional information: 3508
Additional information: 1
Additional information: 1623195648
```

Confirmed: ASMB is trying to attach to **shmid=3508** via `shmat()` and the Linux kernel returns EACCES (Permission Denied).

---

## Step 7 — Verify ASM Is Actually Running

```bash
/u01/app/grid/product/21c/grid/bin/crsctl stat res -t
```

```
ora.DATA.dg    ONLINE  ONLINE  localhost  STABLE
ora.asm        ONLINE  ONLINE  localhost  Started,STABLE
ora.cssd       ONLINE  ONLINE  localhost  STABLE
ora.evmd       ONLINE  ONLINE  localhost  STABLE
```

ASM is running and DATA diskgroup is mounted. This confirms the issue is not ASM being down — it is the **IPC connection mechanism** between the DB instance and ASM that is broken.

```bash
# Also verify DATA diskgroup has space
su - oracle -c "
  export ORACLE_HOME=/u01/app/grid/product/21c/grid
  export ORACLE_SID=+ASM
  export PATH=\$ORACLE_HOME/bin:\$PATH
  sqlplus -s / as sysasm <<'EOF'
  SELECT name, state, total_mb, free_mb FROM v\$asm_diskgroup;
  exit
EOF
"
```

```
NAME   STATE    TOTAL_MB  FREE_MB
DATA   MOUNTED  5120      5056
```

Disk space is fine (98% free). Not the problem.

---

## Step 8 — Check All IPC Shared Memory Segments

```bash
ipcs -m -a
cat /proc/sysvipc/shm
```

```
key         shmid  owner  perms  bytes   nattch
0xb380659c  4      oracle 600    49152   29
```

Only **one** SHM segment exists — **shmid=4**. The shmid referenced in the error (**shmid=3508**) does not exist anywhere in the system. This is the critical clue: `ocssd.bin` is handing out a **stale cached shmid** from a previous ASM boot session. The segment it points to is gone.

Confirming srvctl also fails with the same dead shmid:

```bash
su - oracle -c "
  export ORACLE_HOME=/u01/app/grid/product/21c/grid
  export PATH=\$ORACLE_HOME/bin:\$PATH
  srvctl stop asm -f
"
```

```
ORA-27123: unable to attach to shared memory segment
Linux-x86_64 Error: 13: Permission denied
Additional information: 3508
```

Both srvctl and DBCA's ASMB get the same stale shmid=3508 from `ocssd.bin`. The fix must include restarting `ocssd.bin` to clear its cache.

---

## Step 9 — Check ASM Process Memory Map

```bash
grep "rw-s\|shm" /proc/$(pgrep -f "asm_pmon" | tail -1)/maps | head -5
```

```
60001000-60905000 rw-s ... /dev/shm/ora_+ASM_3011536284_0_0 (deleted)
60c00000-61000000 rw-s ... /dev/shm/ora_+ASM_3011536284_1_0 (deleted)
```

ASM's SGA lives in POSIX shared memory files under `/dev/shm`. These files are created at ASM startup, memory-mapped, then immediately **unlinked** (shown as `deleted`) so they don't persist after ASM exits. The shmid=3508 that `ocssd.bin` was handing out was an IPC reference from a previous ASM boot cycle — the segment had been destroyed but `ocssd.bin` was never told.

---

## Step 10 — Check Oracle Binary Permissions (Key Finding)

```bash
ls -la /u01/app/grid/product/21c/grid/bin/oracle
ls -la /u01/app/oracle/product/21c/dbhome/bin/oracle
```

```
-rwxrwxr-x  1 oracle oinstall  474707776  /u01/app/grid/product/21c/grid/bin/oracle
-rwxrwsr-x  1 oracle dba       498812776  /u01/app/oracle/product/21c/dbhome/bin/oracle
```

**Critical mismatch found:**

| Binary | Group | Setgid | Effective GID when run |
|---|---|---|---|
| Grid home (`oracle`) | `oinstall` | No | oinstall |
| DB home (`oracle`) | `dba` | **Yes** | **dba** |

The Grid home oracle binary was missing the setgid bit and had group `oinstall`. The DB home binary had setgid with group `dba`. This means:

- ASM processes ran with effective GID = `oinstall`
- DB instance's ASMB process ran with effective GID = `dba`

IPC shared memory segments created by ASM (EGID=oinstall) are inaccessible to DB processes running with EGID=dba when group-based permissions are applied. This binary mismatch was the underlying permission root cause that compounded the stale shmid problem.

---

## Fix 1 — Correct Grid Home Oracle Binary Permissions

```bash
# Fix the group and set setgid on the Grid home oracle binary
chown oracle:dba /u01/app/grid/product/21c/grid/bin/oracle
chmod 6755 /u01/app/grid/product/21c/grid/bin/oracle

# Verify — both binaries should now match on group=dba with setgid
ls -la /u01/app/grid/product/21c/grid/bin/oracle
ls -la /u01/app/oracle/product/21c/dbhome/bin/oracle
```

```
-rwsr-sr-x  1 oracle dba  474707776  Grid home oracle   ← now matches
-rwxrwsr-x  1 oracle dba  498812776  DB home oracle
```

---

## Fix 2 — Restart Grid Stack to Clear Stale shmid

The binary permission fix alone is not enough. `ocssd.bin` still holds shmid=3508 in memory. The entire Grid stack must be restarted to force `ocssd.bin` to register fresh IPC info from the newly started ASM.

```bash
# Stop the Grid stack
systemctl stop oracle-ohasd

# Kill any remaining Grid processes
pkill -9 -f "ohasd"
pkill -9 -f "ocssd"
pkill -9 -f "cssdagent"
pkill -9 -f "onmd"
pkill -9 -f "evmd"
pkill -9 -f "oraagent"
sleep 3

# Confirm all processes are gone
ps -ef | grep -E "ohasd|ocssd|cssdagent|evmd|oraagent" | grep -v grep
```

```bash
# Clear all stale IPC segments and socket files
rm -rf /var/tmp/.oracle/* /tmp/.oracle/*
ipcs -m | awk 'NR>3 {print $2}' | xargs -r ipcrm -m
ipcs -s | awk 'NR>3 {print $2}' | xargs -r ipcrm -s

# Verify IPC is clean
ipcs -m   # should be empty
ipcs -s   # should be empty
```

```bash
# Restore capabilities and restart Grid
ldconfig
setcap cap_sys_nice+ep /u01/app/grid/product/21c/grid/bin/cssdagent
setcap cap_sys_nice+ep /u01/app/grid/product/21c/grid/bin/ocssd.bin

systemctl start oracle-ohasd
/u01/app/grid/product/21c/grid/bin/crsctl start has
sleep 30

# Verify Grid is up — CSS and ASM must be ONLINE
/u01/app/grid/product/21c/grid/bin/crsctl stat res -t
```

```
ora.DATA.dg    ONLINE  ONLINE  localhost  STABLE
ora.asm        ONLINE  ONLINE  localhost  Started,STABLE
ora.cssd       ONLINE  ONLINE  localhost  STABLE
ora.evmd       ONLINE  ONLINE  localhost  STABLE
```

```bash
# Verify new shmid is fresh (not 3508) and srvctl can reach ASM
ipcs -m
# Expected: new shmid (e.g. 20), not 3508

su - oracle -c "
  export ORACLE_HOME=/u01/app/grid/product/21c/grid
  export PATH=\$ORACLE_HOME/bin:\$PATH
  srvctl status asm
"
# Expected: ASM is running on localhost
```

---

## Step 11 — Retry DBCA (First Attempt After Fix)

With the IPC fix applied, DBCA was re-run with the original command (`totalMemory=800`).

**This time DBCA progressed further** — past 43% (RMAN restore succeeded), through instance creation, all the way to 100%. But then a new error appeared:

```
100% complete
[FATAL] Error while executing ".../rdbms/admin/utlrp.sql".
Error in Process: .../perl/bin/perl
62% complete
...
0% complete
```

A second distinct failure at 100% during the `utlrp.sql` post-creation step.

---

## Step 12 — Diagnose the utlrp Failure

**Read the DBCA result log:**

```bash
cat /u01/app/oracle/cfgtoollogs/dbca/orcl/orcl11.log
```

```
[FATAL] Error while executing ".../utlrp.sql" ... Error in Process: .../perl
[WARNING] ORA-04031: unable to allocate bytes of shared memory ("","","","")
[WARNING] ORA-65017: seed pluggable database may not be dropped or altered
```

**Check if utlrp SQL itself completed:**

```bash
grep "ERRORS DURING RECOMPILATION" \
  /u01/app/oracle/cfgtoollogs/dbca/orcl/utlrp0.log
```

```
ERRORS DURING RECOMPILATION
----------------------------
          0
```

Zero SQL errors — the SQL completed. The failure was in the Perl `catcon` process that monitors the execution.

**Find the actual catcon error:**

```bash
grep -E "ORA-|error|Error" \
  /u01/app/oracle/cfgtoollogs/dbca/orcl/utlrp_catcon_*.lst \
  | grep -iv "debug\|verbose\|sureunlink" | tail -10
```

```
catcon::print_exec_DB_script_output - ORA-01012: not logged on
catcon::print_exec_DB_script_output - Disconnected
catcon::wait_for_completion - unexpected error in get_instance_status
```

`ORA-01012: not logged on` followed by `Disconnected` — the Oracle instance **crashed** during utlrp, disconnecting the catcon sqlplus session.

**Read the alert log to find why the instance crashed:**

```bash
grep -A2 -B2 "ORA-04031\|abort\|terminate" \
  /u01/app/oracle/diag/rdbms/orcl/orcl/trace/alert_orcl.log | tail -40
```

```
ORA-04031: unable to allocate 319856 bytes of shared memory
  ("shared pool","unknown object","KSK scheduler","KSKQ node")
...
ORA-4031 signalled during: alter pluggable database "PDB$SEED" CLOSE IMMEDIATE
ORA-65017 signalled during: drop pluggable database "PDB$SEED" including datafiles
Shutting down ORACLE instance (abort)
Instance terminated by USER, pid = 15648
Instance shutdown complete
```

**Root cause of utlrp failure:**

`ORA-04031: unable to allocate 319,856 bytes from shared pool` for `KSK scheduler / KSKQ node` — Oracle's internal kernel scheduler. With `totalMemory=800`, the shared pool was too small to simultaneously hold all compiled objects from CDB$ROOT and PDB$SEED. When the shared pool became fully exhausted, Oracle's own internal scheduler could not allocate memory and the instance crashed.

---

## Step 13 — Check ASM Disk Space Before Retry

```bash
su - oracle -c "
  export ORACLE_HOME=/u01/app/grid/product/21c/grid
  export ORACLE_SID=+ASM
  export PATH=\$ORACLE_HOME/bin:\$PATH
  sqlplus -s / as sysasm <<'EOF'
  SELECT name, state, total_mb, free_mb FROM v\$asm_diskgroup;
  SELECT type, count(*), sum(bytes)/1048576 mb
  FROM v\$asm_file GROUP BY type ORDER BY 3 DESC;
  exit
EOF
"
```

```
NAME   STATE    TOTAL_MB  FREE_MB
DATA   MOUNTED  5120      1391     ← only 1.4 GB free!

TYPE        COUNT(*)    MB
DATAFILE    7           2745
ONLINELOG   3           600
TEMPFILE    2           272
CONTROLFILE 1           18
```

DBCA's rollback after the utlrp crash **did not remove the ASM files** it created (3.6 GB of datafiles, redo logs, tempfiles). Only 1.4 GB remains free — not enough to create the database again.

---

## Fix 3 — Reclaim ASM Disk Space

```bash
su - oracle -c "
  export ORACLE_HOME=/u01/app/grid/product/21c/grid
  export ORACLE_SID=+ASM
  export PATH=\$ORACLE_HOME/bin:\$PATH
  sqlplus -s / as sysasm <<'EOF'
  DROP DISKGROUP DATA INCLUDING CONTENTS;
  ALTER SYSTEM SET asm_diskstring='/dev/oracleasm/disks/*' SCOPE=MEMORY;
  CREATE DISKGROUP DATA EXTERNAL REDUNDANCY DISK '/dev/oracleasm/disks/DATA1';
  SELECT name, state, total_mb, free_mb FROM v\$asm_diskgroup;
  exit
EOF
"
```

```
Diskgroup dropped.
System altered.
Diskgroup created.

NAME   STATE    TOTAL_MB  FREE_MB
DATA   MOUNTED  5120      5056    ← full space restored
```

---

## Fix 4 — Retry DBCA with Increased Memory

Increase `totalMemory` from `800` to `1200` to give the shared pool enough room to compile all PDB$SEED objects without exhausting:

```bash
su - oracle -c "
  export ORACLE_HOME=/u01/app/oracle/product/21c/dbhome
  export ORACLE_SID=orcl
  export PATH=\$ORACLE_HOME/bin:\$PATH

  dbca -silent \
    -createDatabase \
    -templateName General_Purpose.dbc \
    -gdbName orcl \
    -sid orcl \
    -characterSet AL32UTF8 \
    -sysPassword Oracle123 \
    -systemPassword Oracle123 \
    -storageType ASM \
    -datafileDestination '+DATA' \
    -recoveryAreaDestination NONE \
    -databaseType MULTIPURPOSE \
    -totalMemory 1200 \
    -emConfiguration NONE \
    -ignorePreReqs
"
```

**Output:**

```
Prepare for db operation
10% complete
Registering database with Oracle Restart
14% complete
Copying database files
43% complete
Creating and starting Oracle instance
45% complete
49% complete
54% complete
58% complete
62% complete
Completing Database Creation
68% complete
71% complete
100% complete
Database creation complete. For details check the logfiles at:
  /u01/app/oracle/cfgtoollogs/dbca/orcl/
```

**Exit code: 0 — Success.**

---

## Step 14 — Verify the Database

```bash
su - oracle -c "
  export ORACLE_HOME=/u01/app/oracle/product/21c/dbhome
  export ORACLE_SID=orcl
  export PATH=\$ORACLE_HOME/bin:\$PATH
  sqlplus -s / as sysdba <<'EOF'
  SELECT instance_name, status FROM v\$instance;
  SELECT name FROM v\$datafile;
  exit
EOF
"
```

```
INSTANCE_NAME    STATUS
orcl             OPEN

NAME
+DATA/ORCL/DATAFILE/system.256.1232143277
+DATA/ORCL/DATAFILE/sysaux.257.1232143331
+DATA/ORCL/DATAFILE/undotbs1.258.1232143357
+DATA/ORCL/C8209F27C6B26005E053362EE80AE60E/DATAFILE/system.265.1232143389
+DATA/ORCL/C8209F27C6B26005E053362EE80AE60E/DATAFILE/sysaux.266.1232143389
+DATA/ORCL/DATAFILE/users.259.1232143359
+DATA/ORCL/C8209F27C6B26005E053362EE80AE60E/DATAFILE/undotbs1.267.1232143389

7 rows selected.
```

Database is **OPEN** with all 7 datafiles on ASM (`+DATA`).

---

## Root Cause Summary

| # | Failure | Root Cause | Fix |
|---|---|---|---|
| 1 | RMAN restore at 43% → rollback | Stale `shmid=3508` cached in `ocssd.bin`; Grid home oracle binary had wrong group (`oinstall`) instead of `dba`, causing IPC permission mismatch with DB home processes | Fix Grid oracle binary: `chown oracle:dba` + `chmod 6755`; full Grid stack restart; clear stale IPC |
| 2 | utlrp.sql crash at 100% → rollback | `totalMemory=800` left shared pool too small; `ORA-04031` (319 KB allocation for KSK scheduler) crashed the instance during PDB$SEED compilation | Increase `totalMemory` from `800` to `1200` |
| 3 | Disk full on retry | DBCA rollback leaves orphaned ASM files (3.6 GB) — datafiles, redo logs, tempfiles | `DROP DISKGROUP DATA INCLUDING CONTENTS` then recreate |
