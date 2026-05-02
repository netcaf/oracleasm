# Oracle 21c + ASM 安装配置指南

## 环境说明
- Hyper-V: Dynamic Memory 4096MB
- OS: Oracle Linux 7.9 (Hyper-V 虚拟机)
- 内核: UEK 5.4.17
- Oracle Grid Infrastructure 21c + Oracle Database 21c
- ASM 磁盘: loop 设备模拟 (5GB)

---

## 一、系统准备

### 1. 创建用户和组
```bash
groupadd -g 54321 oinstall
groupadd -g 54322 dba
groupadd -g 54323 oper
useradd -u 54321 -g oinstall -G dba,oper -m oracle
passwd oracle
```

### 2. 安装依赖包
```bash
yum install -y oracleasm-support oracleasmlib kmod-oracleasm psmisc bc binutils \
  elfutils-libelf elfutils-libelf-devel glibc glibc-devel ksh libaio libaio-devel \
  libX11 libXau libXi libXtst libgcc libstdc++ libstdc++-devel make sysstat
```

### 3. 内核参数
```bash
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
```

### 4. 关闭 SELinux
```bash
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
```

### 5. 禁用 Hyper-V 时间同步 ⚠️
```bash
echo 'blacklist hv_utils' > /etc/modprobe.d/disable-hyperv-timesync.conf
reboot
```

### 6. 修复 /etc/hosts ⚠️
```bash
# 主机名只能映射到一个 IP，否则 ONMD 会崩溃
cat > /etc/hosts << 'EOF'
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.100.11  localhost.localdomain
EOF
```

### 7. 增加 swap
```bash
dd if=/dev/zero of=/swapfile bs=1M count=4096
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
```

### 8. oracle 用户资源限制
```bash
cat >> /etc/security/limits.conf << 'EOF'
oracle soft rtprio 99
oracle hard rtprio 99
oracle soft nofile 65536
oracle hard nofile 65536
oracle soft memlock unlimited
oracle hard memlock unlimited
EOF
```

---

## 二、ASM 磁盘准备

```bash
mkdir -p /opt/asm-disks
dd if=/dev/zero of=/opt/asm-disks/asm_disk1.img bs=1M count=5120
losetup /dev/loop1 /opt/asm-disks/asm_disk1.img

# losetup -l

# 开机自动挂载 loop 设备
echo 'losetup /dev/loop1 /opt/asm-disks/asm_disk1.img' >> /etc/rc.d/rc.local
chmod +x /etc/rc.d/rc.local

# 创建开机自动恢复脚本（ldconfig + capability）⚠️
cat > /etc/rc.d/rc.oracle-grid << 'RCEOF'
#!/bin/bash
ldconfig
setcap cap_sys_nice+ep /u01/app/grid/product/21c/grid/bin/cssdagent
setcap cap_sys_nice+ep /u01/app/grid/product/21c/grid/bin/ocssd.bin
RCEOF
chmod +x /etc/rc.d/rc.oracle-grid
echo '/etc/rc.d/rc.oracle-grid' >> /etc/rc.d/rc.local

# 交互式配置，输入: oracle / dba / y / y
oracleasm configure -i
oracleasm init
oracleasm createdisk DATA1 /dev/loop1
oracleasm listdisks   # 确认输出 DATA1
```

> **重启后恢复**：loop 设备消失，执行：
> ```bash
> losetup /dev/loop1 /opt/asm-disks/asm_disk1.img
> oracleasm scandisks
> oracleasm listdisks   # 确认 DATA1 存在
> ```

---

## 三、安装 Oracle Grid Infrastructure 21c

### 1. 准备目录和配置
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
ldconfig
```

### 2. 解压安装包
```bash
mv /root/LINUX.X64_213000_grid_home.zip /u01/
chown oracle:oinstall /u01/LINUX.X64_213000_grid_home.zip

su - oracle
unzip /u01/LINUX.X64_213000_grid_home.zip -d /u01/app/grid/product/21c/grid
```

### 3. 静默安装
```bash
cd /u01/app/grid/product/21c/grid

./gridSetup.sh -silent \
  -ignorePrereqFailure \
  -responseFile /u01/app/grid/product/21c/grid/install/response/gridsetup.rsp \
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

> **失败恢复**：如报 `INS-30024 invalid Oracle home`，说明有旧配置残留：
> ```bash
> rm -rf /etc/oracle
> rm -f /etc/oraInst.loc
> rm -rf /u01/app/oraInventory/*
> systemctl stop oracle-ohasd 2>/dev/null
> systemctl disable oracle-ohasd 2>/dev/null
> rm -f /etc/systemd/system/oracle-ohasd.service
> systemctl daemon-reload
> # 重新执行步骤 1 和 3
> ```

### 4. 执行 root 脚本
```bash
exit
/u01/app/grid/product/21c/grid/root.sh
```

### 5. 修复二进制文件权限 ⚠️
```bash
# root.sh 执行后 HAS 自动启动，需先停止
/u01/app/grid/product/21c/grid/bin/crsctl stop has -f

cd /u01/app/grid/product/21c/grid/bin
chmod 750 cssdagent ocssd.bin cssdmonitor
setcap cap_sys_nice+ep cssdagent
setcap cap_sys_nice+ep ocssd.bin

# 修复 ocssd.bin owner（原步骤漏了这行）
chown oracle:oinstall ocssd.bin

# 修复 Grid oracle 二进制的组和 setgid 位 ⚠️
# Grid home oracle 默认 group=oinstall、无 setgid，
# 而 DB home oracle 是 group=dba + setgid，
# 导致 ASM 进程 EGID=oinstall，ASMB 进程 EGID=dba，IPC 共享内存跨组无法访问（ORA-27123）
chown oracle:dba /u01/app/grid/product/21c/grid/bin/oracle
chmod 6755 /u01/app/grid/product/21c/grid/bin/oracle

# 验证两个 oracle 二进制的 group 和 setgid 一致
ls -la /u01/app/grid/product/21c/grid/bin/oracle
ls -la /u01/app/oracle/product/21c/dbhome/bin/oracle
# 期望：两者都是 oracle:dba，且都有 setgid 位 (s)

# 验证 cssdagent 能正常加载库
su - oracle -c "/u01/app/grid/product/21c/grid/bin/cssdagent"
# 正常：输出包含 "successfully setting priority"，最后 segfault 属正常
# 失败：报 "error while loading shared libraries: libocr.so" → 执行 ldconfig 再测试
```

### 6. 添加环境变量 ⚠️
```bash
cat >> /u01/app/grid/product/21c/grid/crs/install/s_crsconfig_localhost_env.txt << 'EOF'
ORA_CRS_HOME=/u01/app/grid/product/21c/grid
ORACLE_BASE=/u01/app/oracle
EOF
```

### 7. 启动并验证
```bash
# 步骤一：更新共享库缓存（必须在启动 HAS 之前执行）⚠️
ldconfig

# 步骤二：启动 HAS（两条命令都要执行）
systemctl start oracle-ohasd
/u01/app/grid/product/21c/grid/bin/crsctl start has

# 步骤三：触发 CSS 启动（需等待 2-5 分钟才返回，属正常现象）
# 可在另一个终端监控：tail -f /u01/app/oracle/diag/crs/localhost/crs/trace/alert.log
# 等待看到 CRS-1601: CSSD Reconfiguration complete 说明成功
/u01/app/grid/product/21c/grid/bin/crsctl start res ora.cssd -init

# 步骤四：注册 ASM 资源（只需执行一次）
/u01/app/grid/product/21c/grid/bin/srvctl add asm

# 步骤五：启动 ASM （只需执行一次）
/u01/app/grid/product/21c/grid/bin/srvctl start asm

# 验证，期望: ora.cssd ONLINE, ora.asm ONLINE
/u01/app/grid/product/21c/grid/bin/crsctl stat res -t
```

Oracle Grid Infrastructure 的资源是分层启动的，有两个层次：
┌─────────────────────────────────────────────┐
│         普通资源层（默认层）                  │
│  ora.asm, ora.diskmon, 数据库、监听、VIP…    │
│  由 crsd 进程管理                             │
├─────────────────────────────────────────────┤
│         init 层（底层/初始化层）              │
│  ora.cssd, ora.ctssd, ora.gpnpd, ora.evmd…  │
│  由 ocssd / ohasd 进程管理                   │
└─────────────────────────────────────────────┘

> **CSS 启动失败恢复**：如果 `ora.cssd` 无法变为 ONLINE：
> ```bash
> systemctl stop oracle-ohasd
> pkill -9 -f ohasd
> pkill -9 -f ocssd
> pkill -9 -f cssdagent
> pkill -9 -f onmd
> rm -rf /var/tmp/.oracle/*
> rm -rf /tmp/.oracle/*
> ldconfig
> setcap cap_sys_nice+ep /u01/app/grid/product/21c/grid/bin/cssdagent
> setcap cap_sys_nice+ep /u01/app/grid/product/21c/grid/bin/ocssd.bin
> su - oracle -c "/u01/app/grid/product/21c/grid/bin/cssdagent"  # 验证库加载正常
> systemctl start oracle-ohasd
> /u01/app/grid/product/21c/grid/bin/crsctl start has
> sleep 30
> /u01/app/grid/product/21c/grid/bin/crsctl start res ora.cssd -init
> ```

---

## 四、创建 ASM 磁盘组

## ASM 磁盘组缺失排查与修复

**场景：ASM ONLINE，但 ora.DATA.dg OFFLINE 或不存在**

---

### 诊断

```bash
# OS 层确认磁盘是否可见
oracleasm listdisks
# 无输出则先执行：oracleasm scandisks
```

```sql
-- 进入 ASM
su - oracle
export ORACLE_SID=+ASM
export ORACLE_HOME=/u01/app/grid/product/21c/grid
export PATH=$ORACLE_HOME/bin:$PATH
sqlplus / as sysasm

-- 磁盘搜索路径，VALUE 为空是根本原因
SHOW PARAMETER asm_diskstring;

-- 磁盘是否可见，无行 = 路径未设置
SELECT path, header_status FROM v$asm_disk;

-- 磁盘组状态，MOUNTED=正常 DISMOUNTED=未挂载 无行=不存在
SELECT name, state, total_mb, free_mb FROM v$asm_diskgroup;
```

---

### 修复

```sql
-- Step 1：asm_diskstring 为空时，先临时设置
ALTER SYSTEM SET asm_diskstring='/dev/oracleasm/disks/*' SCOPE=MEMORY;

-- Step 2a：磁盘组存在但未 mount
ALTER DISKGROUP DATA MOUNT;
--如遇到错误，则
SELECT path, header_status, name FROM v$asm_disk;
--如果为PROVISIONED，则可以直接使用， Step 2b

```
`header_status` 的值决定下一步：

| header_status | 含义 | 处理 |
|---|---|---|
| `MEMBER` | 属于某个磁盘组 | 磁盘组元数据问题 |
| `PROVISIONED` | 标记过但未格式化 | 可以重新 CREATE |
| `FOREIGN` | 属于其他磁盘组 | 需要强制清除 |
| `CANDIDATE` | 干净，未使用 | 可以直接 CREATE |

```
-- Step 2b：磁盘组不存在，创建（一次性，永不重复）
CREATE DISKGROUP DATA EXTERNAL REDUNDANCY DISK '/dev/oracleasm/disks/DATA1';

-- 手动挂载磁盘组（磁盘组存在但未 mount 时）
ALTER DISKGROUP DATA MOUNT;
```

---

### 永久固化（一次性配置）

```sql
-- 没有 SPFILE 时先创建
SHOW PARAMETER spfile;
CREATE SPFILE FROM MEMORY;
```
``` bash
# Check the file created
ls -la /u01/app/grid/product/21c/grid/dbs/spfile+ASM.ora
```

```bash
# 重启 ASM 让 SPFILE 生效
srvctl stop diskgroup -g DATA
srvctl stop asm
srvctl start asm
sqlplus / as sysasm
```

```sql
-- Check again
SHOW PARAMETER spfile;

-- 永久写入，重启后自动生效
ALTER SYSTEM SET asm_diskstring='/dev/oracleasm/disks/*' SCOPE=BOTH;
SHOW PARAMETER asm_diskstring;  -- 确认
```

```bash
# 配置开机自动 scandisks
oracleasm configure -i    # SCANBOOT 选 y

# 确认修复
/u01/app/grid/product/21c/grid/bin/crsctl stat res -t        # ora.DATA.dg = ONLINE ✅
```

---

> **重启后磁盘组丢失恢复**：
> ```bash
> losetup -a                  # 确认 loop 设备存在
> oracleasm listdisks         # 确认 DATA1 存在
> sqlplus / as sysasm
> ALTER SYSTEM SET asm_diskstring='/dev/oracleasm/disks/*' SCOPE=MEMORY;
> CREATE DISKGROUP DATA EXTERNAL REDUNDANCY DISK '/dev/oracleasm/disks/DATA1';
> exit
> ```

---

## 五、安装 Oracle Database 21c

### 1. 准备目录并解压
```bash
mkdir -p /u01/app/oracle/product/21c/dbhome
chown -R oracle:oinstall /u01/app/oracle

mv /root/LINUX.X64_213000_db_home.zip /u01/
chown oracle:oinstall /u01/LINUX.X64_213000_db_home.zip

su - oracle
unzip /u01/LINUX.X64_213000_db_home.zip -d /u01/app/oracle/product/21c/dbhome
```

### 2. 静默安装
```bash
export ORACLE_HOME=/u01/app/oracle/product/21c/dbhome
export PATH=$ORACLE_HOME/bin:$PATH
cd $ORACLE_HOME

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

### 3. 执行 root 脚本
```bash
exit
/u01/app/oracle/product/21c/dbhome/root.sh
```

---

## 六、创建数据库

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
  -totalMemory 1200 \
  -emConfiguration NONE \
  -ignorePreReqs
```

> **dbca 失败恢复**：dbca 失败后按以下顺序处理：
>
> **1. 确认 Grid oracle 二进制权限正确（一次性）**
> ```bash
> ls -la /u01/app/grid/product/21c/grid/bin/oracle
> # 若 group 不是 dba 或没有 setgid，执行：
> chown oracle:dba /u01/app/grid/product/21c/grid/bin/oracle
> chmod 6755 /u01/app/grid/product/21c/grid/bin/oracle
> ```
>
> **2. 重启 Grid 栈以清除 ocssd.bin 中缓存的旧 shmid**
> ```bash
> systemctl stop oracle-ohasd
> pkill -9 -f "ohasd"; pkill -9 -f "ocssd"; pkill -9 -f "cssdagent"
> pkill -9 -f "onmd";  pkill -9 -f "evmd";  pkill -9 -f "oraagent"
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
> # 确认 ora.cssd / ora.asm / ora.DATA.dg 均为 ONLINE
> ```
>
> **3. 清理 ASM 残留文件（dbca 回滚不会自动删除 ASM 中的数据文件）**
> ```bash
> su - oracle -c "
>   export ORACLE_HOME=/u01/app/grid/product/21c/grid
>   export ORACLE_SID=+ASM
>   export PATH=\$ORACLE_HOME/bin:\$PATH
>   sqlplus -s / as sysasm <<'EOF'
>   DROP DISKGROUP DATA INCLUDING CONTENTS;
>   ALTER SYSTEM SET asm_diskstring='/dev/oracleasm/disks/*' SCOPE=MEMORY;
>   CREATE DISKGROUP DATA EXTERNAL REDUNDANCY DISK '/dev/oracleasm/disks/DATA1';
>   SELECT name, state, total_mb, free_mb FROM v\$asm_diskgroup;
>   exit
> EOF
> "
> # 确认 FREE_MB 恢复到约 5056
> ```
>
> **4. 重新建库**
> ```bash
> export ORACLE_HOME=/u01/app/oracle/product/21c/dbhome
> export ORACLE_SID=orcl
> export PATH=$ORACLE_HOME/bin:$PATH
> dbca -silent \
>   -createDatabase \
>   -templateName General_Purpose.dbc \
>   -gdbName orcl -sid orcl \
>   -characterSet AL32UTF8 \
>   -sysPassword Oracle123 \
>   -systemPassword Oracle123 \
>   -storageType ASM \
>   -datafileDestination '+DATA' \
>   -recoveryAreaDestination NONE \
>   -databaseType MULTIPURPOSE \
>   -totalMemory 1200 \
>   -emConfiguration NONE \
>   -ignorePreReqs
> ```

### 验证
```bash
export ORACLE_HOME=/u01/app/oracle/product/21c/dbhome
export ORACLE_SID=orcl
export PATH=$ORACLE_HOME/bin:$PATH

sqlplus / as sysdba
```

```sql
SELECT instance_name, status FROM v$instance;
SELECT name FROM v$datafile;
exit
```

---

## 七、重启后启动流程

每次系统重启后，需按以下顺序手动恢复：

### 第一步：恢复 ASM 磁盘
```bash
# 确认 loop 设备是否存在（rc.local 应自动执行，如未执行则手动运行）
losetup -a
# 如果没有输出，手动绑定
losetup /dev/loop1 /opt/asm-disks/asm_disk1.img

# 扫描并确认磁盘
oracleasm scandisks
oracleasm listdisks   # 确认输出 DATA1
```

### 第二步：更新库缓存和权限
```bash
# rc.oracle-grid 应自动执行，如未执行则手动运行
ldconfig
setcap cap_sys_nice+ep /u01/app/grid/product/21c/grid/bin/cssdagent
setcap cap_sys_nice+ep /u01/app/grid/product/21c/grid/bin/ocssd.bin

# 验证库加载正常
su - oracle -c "/u01/app/grid/product/21c/grid/bin/cssdagent"
# 正常：输出包含 "successfully setting priority"
```

### 第三步：启动 Grid 和 CSS
```bash
systemctl start oracle-ohasd
/u01/app/grid/product/21c/grid/bin/crsctl start has

# 触发 CSS 启动（等待 2-5 分钟）
/u01/app/grid/product/21c/grid/bin/crsctl start res ora.cssd -init

# 验证 CSS 和 ASM 状态
/u01/app/grid/product/21c/grid/bin/crsctl stat res -t
# 期望: ora.cssd ONLINE, ora.asm ONLINE
```

### 第三步补充：确认磁盘组已挂载

`ora.DATA.dg` 是 CRS 自动资源——只有当 ASM 实例内部已将 DATA 磁盘组 MOUNT 后，它才会出现在 `crsctl stat res -t` 中。ASM 进程 ONLINE 并不等于磁盘组已挂载。

```bash
su - oracle -c "
  export ORACLE_HOME=/u01/app/grid/product/21c/grid
  export ORACLE_SID=+ASM
  export PATH=\$ORACLE_HOME/bin:\$PATH
  sqlplus -s / as sysasm <<'EOF'
  SELECT name, state FROM v\$asm_diskgroup;
  exit
EOF
"
```

- 如果 `state=MOUNTED` → 正常，`ora.DATA.dg` 应已出现在 crsctl 输出中
- 如果 `state=DISMOUNTED` → SPFILE 中 `asm_diskstring` 为空，执行：
  ```bash
  # 临时恢复（重启后再次失效，需先确保 SPFILE 已设置 asm_diskstring）
  sqlplus / as sysasm
  ALTER SYSTEM SET asm_diskstring='/dev/oracleasm/disks/*' SCOPE=MEMORY;
  ALTER DISKGROUP DATA MOUNT;
  ```
- 如果无行返回 → 磁盘组不存在，先确认 loop 设备和磁盘标签，再参考第四节创建

### 第四步：启动数据库
```bash
export ORACLE_HOME=/u01/app/oracle/product/21c/dbhome
export ORACLE_SID=orcl
export PATH=$ORACLE_HOME/bin:$PATH

sqlplus / as sysdba
```

```sql
startup
exit
```

### 验证整体状态
```bash
/u01/app/grid/product/21c/grid/bin/crsctl stat res -t
# 期望输出:
# ora.asm    ONLINE ONLINE
# ora.cssd   ONLINE ONLINE
# ora.evmd   ONLINE ONLINE
```

---

## 关键问题汇总

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| CSS 无法启动 | Hyper-V 时间同步干扰 | blacklist hv_utils |
| ONMD 崩溃 | /etc/hosts 主机名映射冲突 | 每个主机名只映射一个 IP |
| cssdagent 启动失败 | 权限不足 | chmod 750 + setcap cap_sys_nice+ep |
| libocr.so 找不到 | ldconfig 缓存未更新 | 启动前执行 ldconfig |
| ORA_CRS_HOME 找不到 | 环境变量未传递 | 写入 s_crsconfig_env.txt |
| ASM 找不到磁盘 | diskstring 未设置 | ALTER SYSTEM SET asm_diskstring |
| ora.DATA.dg 不出现在 crsctl 输出 | ASM 进程 ONLINE 但磁盘组未挂载（asm_diskstring 为空或 loop 设备不存在） | 在 ASM 内执行 ALTER SYSTEM SET asm_diskstring + ALTER DISKGROUP DATA MOUNT；挂载后自动出现 |
| ASMB ORA-27123 Permission denied | Grid oracle 二进制 group=oinstall 无 setgid，EGID 与 DB home 不一致，IPC 共享内存跨组拒绝访问 | chown oracle:dba + chmod 6755 Grid home oracle；重启 Grid 栈清除 ocssd.bin 的旧 shmid 缓存 |
| utlrp.sql ORA-04031 实例崩溃 | totalMemory=800 时 shared pool 不足，KSK 内部调度器无法分配内存，CDB$ROOT + PDB$SEED 编译耗尽共享池 | totalMemory 改为 1200 |
| dbca 失败后空间不足 | dbca 回滚不删除 ASM 中已写入的数据文件/日志/临时文件（可达 3.6 GB） | DROP DISKGROUP DATA INCLUDING CONTENTS 后重建 |
