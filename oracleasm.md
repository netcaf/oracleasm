# Oracle 21c + ASM Installation Guide

## Environment
- Hyper-V Dynamic Memory: 4096 MB
- OS: Oracle Linux 7.9
- Kernel: UEK 5.4.17
- Oracle Grid Infrastructure 21c + Oracle Database 21c
- ASM disk: loop device simulation (5 GB)

---

## I. System Preparation

Run as **root**. Reboot once at the end.

```bash
# 1. Users and groups
groupadd -g 54321 oinstall
groupadd -g 54322 dba
groupadd -g 54323 oper
useradd -u 54321 -g oinstall -G dba,oper -m oracle
passwd oracle

# 2. Dependencies
yum install -y oracleasm-support oracleasmlib kmod-oracleasm psmisc bc binutils \
  elfutils-libelf elfutils-libelf-devel glibc glibc-devel ksh libaio libaio-devel \
  libX11 libXau libXi libXtst libgcc libstdc++ libstdc++-devel make sysstat

# 3. Kernel parameters
cat >> /etc/sysctl.conf << 'EOF'
fs.aio-max-nr = 1048576
fs.file-max = 6815744
kernel.shmall = 2097152
kernel.shmmax = 4294967295
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
EOF
sysctl -p

# 4. SELinux
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config

# 5. Disable Hyper-V time sync — requires reboot; CSS depends on a stable clock ⚠️
echo 'blacklist hv_utils' > /etc/modprobe.d/disable-hyperv-timesync.conf

# 6. /etc/hosts — hostname must map to exactly one IP, or ONMD crashes ⚠️
cat > /etc/hosts << 'EOF'
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.100.11  localhost.localdomain
EOF

# 7. oracle user resource limits
cat >> /etc/security/limits.conf << 'EOF'
oracle soft rtprio 99
oracle hard rtprio 99
oracle soft nofile 65536
oracle hard nofile 65536
oracle soft memlock unlimited
oracle hard memlock unlimited
EOF

reboot
```

After reboot, run as **root**:

```bash
# 8. oracle user environment
cat >> /home/oracle/.bash_profile << 'EOF'

export ORACLE_BASE=/u01/app/oracle
export GRID_HOME=/u01/app/grid/product/21c/grid
export DB_HOME=/u01/app/oracle/product/21c/dbhome
export ORACLE_HOME=$DB_HOME
export ORACLE_SID=orcl
export PATH=$DB_HOME/bin:$GRID_HOME/bin:$PATH

function asmenv { export ORACLE_HOME=$GRID_HOME; export ORACLE_SID=+ASM; }
function dbenv  { export ORACLE_HOME=$DB_HOME;   export ORACLE_SID=orcl; }
EOF
```

After this, `su - oracle` opens a DB context by default.
- `sqlplus / as sysdba` — works directly
- `asmenv` then `sqlplus / as sysasm` — switch to ASM context
- `dbenv` — switch back to DB context

---

## II. ASM Disk Preparation

Run as **root**.

```bash
mkdir -p /opt/asm-disks
dd if=/dev/zero of=/opt/asm-disks/asm_disk1.img bs=1M count=5120
losetup /dev/loop1 /opt/asm-disks/asm_disk1.img

# Auto-mount loop device on boot
echo 'losetup /dev/loop1 /opt/asm-disks/asm_disk1.img' >> /etc/rc.d/rc.local
chmod +x /etc/rc.d/rc.local

# Auto-restore ldconfig and setcap on boot
cat > /etc/rc.d/rc.oracle-grid << 'RCEOF'
#!/bin/bash
ldconfig
setcap cap_sys_nice+ep /u01/app/grid/product/21c/grid/bin/cssdagent
setcap cap_sys_nice+ep /u01/app/grid/product/21c/grid/bin/ocssd.bin
RCEOF
chmod +x /etc/rc.d/rc.oracle-grid
echo '/etc/rc.d/rc.oracle-grid' >> /etc/rc.d/rc.local

# Interactive — enter: oracle / dba / y / y
oracleasm configure -i
oracleasm init
oracleasm createdisk DATA1 /dev/loop1
oracleasm listdisks   # confirm DATA1 is listed
```

> **After reboot** — if DATA1 is missing, see Section VII Step 1.

---

## III. Install Oracle Grid Infrastructure 21c

### 1. Prepare directories (root)
```bash
mkdir -p /u01/app/grid/product/21c/grid
mkdir -p /u01/app/oraInventory
chown -R oracle:oinstall /u01
chmod -R 775 /u01

cat > /etc/oraInst.loc << 'EOF'
inventory_loc=/u01/app/oraInventory
inst_group=oinstall
EOF

echo "/u01/app/grid/product/21c/grid/lib" > /etc/ld.so.conf.d/oracle-grid.conf
```

### 2. Extract installer (oracle)
```bash
mv /root/LINUX.X64_213000_grid_home.zip /u01/
chown oracle:oinstall /u01/LINUX.X64_213000_grid_home.zip

su - oracle
unzip /u01/LINUX.X64_213000_grid_home.zip -d $GRID_HOME
```

### 3. Silent install (oracle)
```bash
cd $GRID_HOME

./gridSetup.sh -silent \
  -ignorePrereqFailure \
  -responseFile $GRID_HOME/install/response/gridsetup.rsp \
  oracle.install.option=HA_CONFIG \
  ORACLE_BASE=/u01/app/oracle \
  oracle.install.asm.OSDBA=dba \
  oracle.install.asm.OSOPER=oper \
  oracle.install.asm.OSASM=dba \
  oracle.install.asm.diskGroup.name=DATA \
  oracle.install.asm.diskGroup.redundancy=EXTERNAL \
  oracle.install.asm.diskGroup.disks=/dev/oracleasm/disks/DATA1 \
  oracle.install.asm.diskGroup.diskDiscoveryString=/dev/oracleasm/disks/* \
  oracle.install.asm.monitorPassword=Oracle123 \
  oracle.install.asm.SYSASMPassword=Oracle123 \
  oracle.install.asm.SYSPassword=Oracle123 \
  INVENTORY_LOCATION=/u01/app/oraInventory
```

> **Recovery** — if `INS-30024 invalid Oracle home` (stale config from a previous attempt):
> ```bash
> rm -rf /etc/oracle /etc/oraInst.loc /u01/app/oraInventory/*
> systemctl stop oracle-ohasd 2>/dev/null; systemctl disable oracle-ohasd 2>/dev/null
> rm -f /etc/systemd/system/oracle-ohasd.service; systemctl daemon-reload
> # Re-run steps 1 and 3
> ```

### 4. Run root script (root)
```bash
exit   # back to root
/u01/app/grid/product/21c/grid/root.sh
```

### 5. Fix binary permissions (root) ⚠️
```bash
# root.sh auto-starts HAS — stop it first
/u01/app/grid/product/21c/grid/bin/crsctl stop has -f

cd /u01/app/grid/product/21c/grid/bin
chmod 750 cssdagent ocssd.bin cssdmonitor
setcap cap_sys_nice+ep cssdagent
setcap cap_sys_nice+ep ocssd.bin

# Grid oracle binary must match DB home: same group + setgid, or ASMB IPC fails (ORA-27123)
chown oracle:dba oracle
chmod 6755 oracle

# Verify: both oracle binaries show oracle:dba with setgid (s)
ls -la /u01/app/grid/product/21c/grid/bin/oracle
ls -la /u01/app/oracle/product/21c/dbhome/bin/oracle

# Verify library loading
su - oracle -c "$GRID_HOME/bin/cssdagent"
# OK:   output contains "successfully setting priority"; trailing segfault is normal
# Fail: "error while loading shared libraries" → run ldconfig and retry
```

### 6. Add environment variables (root) ⚠️
```bash
cat >> /u01/app/grid/product/21c/grid/crs/install/s_crsconfig_localhost_env.txt << 'EOF'
ORA_CRS_HOME=/u01/app/grid/product/21c/grid
ORACLE_BASE=/u01/app/oracle
EOF
```

### 7. Start Grid stack (root)
```bash
ldconfig   # must run before starting HAS

systemctl start oracle-ohasd
/u01/app/grid/product/21c/grid/bin/crsctl start has

# Start CSS — takes 2-5 minutes
# Monitor: tail -f /u01/app/oracle/diag/crs/localhost/crs/trace/alert.log
#          wait for: CRS-1601: CSSD Reconfiguration complete
/u01/app/grid/product/21c/grid/bin/crsctl start res ora.cssd -init

# Register and start ASM (one-time only)
/u01/app/grid/product/21c/grid/bin/srvctl add asm
/u01/app/grid/product/21c/grid/bin/srvctl start asm

# Verify: ora.cssd ONLINE, ora.asm ONLINE, ora.DATA.dg ONLINE
/u01/app/grid/product/21c/grid/bin/crsctl stat res -t
```

Grid resources run in two layers:
```
ohasd (systemd)
├── init layer:   ora.cssd, ora.evmd, ora.ctssd  ← crsctl start res ... -init
└── normal layer: ora.asm, ora.DATA.dg, databases ← srvctl start asm
```

> **CSS start failure recovery**:
> ```bash
> systemctl stop oracle-ohasd
> pkill -9 -f "ohasd|ocssd|cssdagent|onmd"
> rm -rf /var/tmp/.oracle/* /tmp/.oracle/*
> ldconfig
> setcap cap_sys_nice+ep /u01/app/grid/product/21c/grid/bin/cssdagent
> setcap cap_sys_nice+ep /u01/app/grid/product/21c/grid/bin/ocssd.bin
> systemctl start oracle-ohasd
> /u01/app/grid/product/21c/grid/bin/crsctl start has
> sleep 30
> /u01/app/grid/product/21c/grid/bin/crsctl start res ora.cssd -init
> ```

---

## IV. ASM Diskgroup Configuration

Grid installation already created the DATA diskgroup. This section persists `asm_diskstring` to the SPFILE so it survives reboots.

Run as **oracle**:

```bash
su - oracle
asmenv
sqlplus / as sysasm
```

```sql
-- Confirm diskgroup is mounted
SELECT name, state, total_mb, free_mb FROM v$asm_diskgroup;

-- Persist diskstring (one-time)
ALTER SYSTEM SET asm_diskstring='/dev/oracleasm/disks/*' SCOPE=SPFILE;
```

### Troubleshooting: ora.DATA.dg not appearing

`ora.DATA.dg` is a CRS auto-resource — it appears automatically once the diskgroup is mounted. No manual registration needed.

In the same sqlplus session, diagnose:

```sql
SHOW PARAMETER asm_diskstring;           -- empty value is the root cause
SELECT path, header_status FROM v$asm_disk;
SELECT name, state FROM v$asm_diskgroup;
```

| Diskgroup state | Action |
|---|---|
| `MOUNTED` | Normal — `ora.DATA.dg` should already appear in crsctl |
| `DISMOUNTED` | `ALTER SYSTEM SET asm_diskstring='/dev/oracleasm/disks/*' SCOPE=MEMORY;` then `ALTER DISKGROUP DATA MOUNT;` |
| No rows (missing) | Set `asm_diskstring` first (same as above), confirm disk appears in `v$asm_disk`, then `CREATE DISKGROUP` |

`header_status` determines whether CREATE is safe:

| header_status | Action |
|---|---|
| `CANDIDATE` / `PROVISIONED` | `CREATE DISKGROUP DATA EXTERNAL REDUNDANCY DISK '/dev/oracleasm/disks/DATA1';` |
| `MEMBER` | Diskgroup metadata remains — DROP first, then recreate |

---

## V. Install Oracle Database 21c

### 1. Prepare directories (root) and extract (oracle)
```bash
mkdir -p /u01/app/oracle/product/21c/dbhome
chown -R oracle:oinstall /u01/app/oracle

mv /root/LINUX.X64_213000_db_home.zip /u01/
chown oracle:oinstall /u01/LINUX.X64_213000_db_home.zip

su - oracle
unzip /u01/LINUX.X64_213000_db_home.zip -d $DB_HOME
```

### 2. Silent install (oracle)
```bash
cd $DB_HOME

./runInstaller -silent \
  -ignorePrereqFailure \
  oracle.install.option=INSTALL_DB_SWONLY \
  ORACLE_BASE=/u01/app/oracle \
  oracle.install.db.InstallEdition=EE \
  oracle.install.db.OSDBA_GROUP=dba \
  oracle.install.db.OSOPER_GROUP=oper \
  oracle.install.db.OSBACKUPDBA_GROUP=dba \
  oracle.install.db.OSDGDBA_GROUP=dba \
  oracle.install.db.OSKMDBA_GROUP=dba \
  oracle.install.db.OSRACDBA_GROUP=dba \
  INVENTORY_LOCATION=/u01/app/oraInventory
```

### 3. Run root script (root)
```bash
exit   # back to root
/u01/app/oracle/product/21c/dbhome/root.sh
```

---

## VI. Create Database

Run as **oracle**:

```bash
su - oracle

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
```

> **dbca failure recovery**:
>
> **1. Verify Grid oracle binary permissions (one-time)**
> ```bash
> ls -la /u01/app/grid/product/21c/grid/bin/oracle
> # If group is not dba or setgid is missing:
> chown oracle:dba /u01/app/grid/product/21c/grid/bin/oracle
> chmod 6755 /u01/app/grid/product/21c/grid/bin/oracle
> ```
>
> **2. Restart Grid stack to clear stale shmid cached in ocssd.bin (root)**
> ```bash
> systemctl stop oracle-ohasd
> pkill -9 -f "ohasd|ocssd|cssdagent|onmd|evmd|oraagent"
> sleep 3
> rm -rf /var/tmp/.oracle/* /tmp/.oracle/*
> ipcs -m | awk 'NR>3 {print $2}' | xargs -r ipcrm -m
> ipcs -s | awk 'NR>3 {print $2}' | xargs -r ipcrm -s
> ldconfig
> setcap cap_sys_nice+ep /u01/app/grid/product/21c/grid/bin/cssdagent
> setcap cap_sys_nice+ep /u01/app/grid/product/21c/grid/bin/ocssd.bin
> systemctl start oracle-ohasd
> /u01/app/grid/product/21c/grid/bin/crsctl start has
> sleep 30
> /u01/app/grid/product/21c/grid/bin/crsctl stat res -t
> # Confirm: ora.cssd / ora.asm / ora.DATA.dg all ONLINE
> ```
>
> **3. Reclaim ASM disk space — dbca rollback does not remove ASM files (oracle)**
> ```bash
> su - oracle
> asmenv
> sqlplus / as sysasm
> ```
> ```sql
> DROP DISKGROUP DATA INCLUDING CONTENTS;
> ALTER SYSTEM SET asm_diskstring='/dev/oracleasm/disks/*' SCOPE=MEMORY;
> CREATE DISKGROUP DATA EXTERNAL REDUNDANCY DISK '/dev/oracleasm/disks/DATA1';
> SELECT name, state, total_mb, free_mb FROM v$asm_diskgroup;   -- FREE_MB should be ~5056
> exit
> ```

### Verify

Run as **root**:

```bash
# 1. Grid stack — CSS, ASM, and DATA diskgroup must all be ONLINE
/u01/app/grid/product/21c/grid/bin/crsctl stat res -t

# 2. Database instance and datafiles
su - oracle -c 'sqlplus -s / as sysdba <<EOF
SELECT instance_name, status FROM v\$instance;
SELECT name, open_mode FROM v\$database;
SELECT name FROM v\$datafile;
exit
EOF'

# 3. ASM diskgroup space (override to Grid home — profile defaults to DB home)
su - oracle -c 'ORACLE_HOME=/u01/app/grid/product/21c/grid ORACLE_SID=+ASM sqlplus -s / as sysasm <<EOF
SELECT name, state, total_mb, free_mb FROM v\$asm_diskgroup;
exit
EOF'
```

---

## VII. Post-reboot Startup

### Step 1: Restore ASM disk (root)
```bash
# rc.local runs this automatically; run manually if it did not
losetup /dev/loop1 /opt/asm-disks/asm_disk1.img
oracleasm scandisks
oracleasm listdisks   # confirm DATA1 is listed
```

### Step 2: Restore library cache and capabilities (root)
```bash
# rc.oracle-grid runs this automatically; run manually if it did not
ldconfig
setcap cap_sys_nice+ep /u01/app/grid/product/21c/grid/bin/cssdagent
setcap cap_sys_nice+ep /u01/app/grid/product/21c/grid/bin/ocssd.bin

su - oracle -c "$GRID_HOME/bin/cssdagent"
# OK: output contains "successfully setting priority"
```

### Step 3: Start Grid and CSS (root)
```bash
systemctl start oracle-ohasd
/u01/app/grid/product/21c/grid/bin/crsctl start has
/u01/app/grid/product/21c/grid/bin/crsctl start res ora.cssd -init   # wait 2-5 minutes

/u01/app/grid/product/21c/grid/bin/crsctl stat res -t
# Expected: ora.cssd ONLINE, ora.asm ONLINE, ora.DATA.dg ONLINE
```

> If `ora.DATA.dg` does not appear, see Section IV troubleshooting.

### Step 4: Start database (oracle)
```bash
su - oracle
sqlplus / as sysdba
```
```sql
startup
exit
```

---

## VIII. Adding an ASM Disk

### Option A — Expand existing diskgroup (DATA)

Run as **root**:

```bash
# Create and bind the new disk image
dd if=/dev/zero of=/opt/asm-disks/asm_disk2.img bs=1M count=5120
losetup /dev/loop2 /opt/asm-disks/asm_disk2.img

# Persist loop device across reboots
echo 'losetup /dev/loop2 /opt/asm-disks/asm_disk2.img' >> /etc/rc.d/rc.local

# Label the disk for ASM
oracleasm createdisk DATA2 /dev/loop2
oracleasm listdisks   # confirm DATA2 is listed
```

Run as **oracle**:

```bash
su - oracle
asmenv
sqlplus / as sysasm
```

```sql
-- Add disk to existing diskgroup; ASM rebalances automatically
ALTER DISKGROUP DATA ADD DISK '/dev/oracleasm/disks/DATA2';

-- 1. Wait for rebalance to complete (no rows = done)
SELECT group_number, operation, state, est_minutes FROM v$asm_operation;

-- 2. Confirm new disk is MEMBER/NORMAL in the group
SELECT path, name, state, header_status FROM v$asm_disk WHERE group_number > 0;

-- 3. Confirm total capacity increased
SELECT name, state, total_mb, free_mb FROM v$asm_diskgroup;
```

All three checks must pass before the disk is confirmed working:
- `v$asm_operation` — no rows (rebalance complete)
- `v$asm_disk` — new disk shows `state=NORMAL`, `header_status=MEMBER`
- `v$asm_diskgroup` — `total_mb` reflects both disks combined

### Option B — Create a new diskgroup (e.g., FRA)

Same OS steps as Option A (use a different label, e.g., `FRA1`), then:

```sql
CREATE DISKGROUP FRA EXTERNAL REDUNDANCY DISK '/dev/oracleasm/disks/FRA1';
SELECT name, state, total_mb, free_mb FROM v$asm_diskgroup;
```

### Option C — Remove a disk from a diskgroup

**Pre-check** — remaining disks must have enough free space to absorb the data from the disk being removed.

Run as **oracle** (`asmenv` + `sqlplus / as sysasm`):

```sql
-- Check disk layout: ensure remaining disks' free_mb >= used_mb of disk to remove
-- used_mb = total_mb - free_mb of the disk being dropped
SELECT path, name, total_mb, free_mb, total_mb - free_mb used_mb
FROM v$asm_disk WHERE group_number > 0;
```

**Drop the disk** — use the `NAME` column value from `v$asm_disk`, not the path or oracleasm label:

```sql
-- ASM rebalances data off the disk automatically before removing it
ALTER DISKGROUP DATA DROP DISK 'DATA_0001';

-- Monitor rebalance (no rows = complete, safe to proceed)
SELECT group_number, operation, state, est_minutes FROM v$asm_operation;

-- Confirm disk is gone and capacity reduced
SELECT path, name, state, header_status FROM v$asm_disk WHERE group_number > 0;
SELECT name, state, total_mb, free_mb FROM v$asm_diskgroup;
```

**OS cleanup** — only after ASM confirms the disk is fully removed:

```bash
oracleasm deletedisk DATA2
losetup -d /dev/loop2
rm -f /opt/asm-disks/asm_disk2.img

# Remove from rc.local
sed -i '/asm_disk2/d' /etc/rc.d/rc.local
```

> **Warning** — never detach the loop device or delete the image file before rebalance completes. ASM would lose data mid-transfer.
>
> **FORCE option** — `ALTER DISKGROUP DATA DROP DISK 'DATA_0001' FORCE;` skips rebalance and is only safe when the disk has already physically failed and its data is lost. Do not use on a healthy disk.
>
> **Last disk** — `DROP DISK` cannot be used on the last disk in a group. Use `DROP DISKGROUP DATA INCLUDING CONTENTS;` instead.

### Notes

- **`asm_diskstring` already covers new disks** — the wildcard `/dev/oracleasm/disks/*` discovers any newly labeled disk automatically. No change needed.
- **Rebalancing** — `ALTER DISKGROUP ADD` triggers automatic rebalancing in the background. On 5 GB loop devices it completes in seconds; on large real disks it can be I/O intensive.
- **EXTERNAL REDUNDANCY** — adding a disk to DATA increases capacity only, not redundancy. Mirroring requires NORMAL (2 failure groups) or HIGH (3 failure groups) redundancy, which must be set at `CREATE DISKGROUP` time.

---

## Known Issues

| Issue | Cause | Fix |
|---|---|---|
| CSS fails to start | Hyper-V time sync interference | Blacklist `hv_utils` (Section I step 5) |
| ONMD crashes | Hostname maps to multiple IPs | One IP per hostname in `/etc/hosts` |
| `cssdagent` fails to start | Missing permissions | `chmod 750` + `setcap cap_sys_nice+ep` |
| `libocr.so` not found | `ldconfig` cache stale | Run `ldconfig` before starting HAS |
| `ORA_CRS_HOME` not found | Env var not passed to CRS | Add to `s_crsconfig_localhost_env.txt` |
| `ora.DATA.dg` not appearing | Diskgroup not mounted | Set `asm_diskstring` + `ALTER DISKGROUP DATA MOUNT` — see Section IV |
| ASMB `ORA-27123` permission denied | Grid `oracle` binary wrong group/setgid; EGID mismatch blocks IPC | `chown oracle:dba` + `chmod 6755`; restart Grid stack to clear stale shmid |
| `utlrp.sql` `ORA-04031` crash | Shared pool exhausted during PDB$SEED compilation | Increase `totalMemory` to 1200 |
| Disk space exhausted after dbca failure | dbca rollback leaves orphaned ASM files (up to 3.6 GB) | `DROP DISKGROUP DATA INCLUDING CONTENTS` — see Section VI recovery step 3 |
