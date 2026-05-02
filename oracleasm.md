# Oracle 21c + ASM 安装配置指南

## 环境说明
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
# /u01/app/grid/product/21c/grid/bin/crsctl start res ora.cssd -init

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

```bash
su - oracle
export ORACLE_HOME=/u01/app/grid/product/21c/grid
export ORACLE_SID=+ASM
export PATH=$ORACLE_HOME/bin:$PATH

sqlplus / as sysasm
```

```sql
-- --------------------------------------------
-- 一次性命令（只执行一次，永不重复）
-- --------------------------------------------

-- 创建磁盘组，相当于格式化磁盘
CREATE DISKGROUP DATA EXTERNAL REDUNDANCY DISK '/dev/oracleasm/disks/DATA1';


-- --------------------------------------------
-- 永久配置（只配置一次，写入 SPFILE 持久化）
-- --------------------------------------------

-- 告诉 ASM 去哪找磁盘，重启后依然生效
ALTER SYSTEM SET asm_diskstring='/dev/oracleasm/disks/*' SCOPE=SPFILE;


-- --------------------------------------------
-- 临时补救命令（scandisks 晚于 ASM 启动时才用）
-- --------------------------------------------

-- 临时让 ASM 重新扫描磁盘路径，重启后失效，第一次为空需要扫描操作。
ALTER SYSTEM SET asm_diskstring='/dev/oracleasm/disks/*' SCOPE=MEMORY;

-- 手动挂载磁盘组（磁盘组存在但未 mount 时）
ALTER DISKGROUP DATA MOUNT;


-- --------------------------------------------
-- 查询命令（随时可用）
-- --------------------------------------------

-- 查看磁盘组状态
SELECT name, state, total_mb, free_mb FROM v$asm_diskgroup;
```

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
  -totalMemory 800 \
  -emConfiguration NONE \
  -ignorePreReqs
```

> **dbca 失败恢复**：如果 dbca 失败后 ASM 空间不足，清理残留文件重建：
> ```bash
> export ORACLE_HOME=/u01/app/grid/product/21c/grid
> export ORACLE_SID=+ASM
> export PATH=$ORACLE_HOME/bin:$PATH
> sqlplus / as sysasm
> DROP DISKGROUP DATA INCLUDING CONTENTS;
> exit
> sqlplus / as sysasm
> ALTER SYSTEM SET asm_diskstring='/dev/oracleasm/disks/*' SCOPE=MEMORY;
> CREATE DISKGROUP DATA EXTERNAL REDUNDANCY DISK '/dev/oracleasm/disks/DATA1';
> exit
> # 重新建库
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
>   -totalMemory 800 \
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
| 建库内存不足 | SGA 最小 560MB | totalMemory 800 + swap |
| dbca 失败空间不足 | 上次失败残留文件 | DROP DISKGROUP 清理后重建 |
