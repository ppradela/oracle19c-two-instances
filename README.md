# 🏛️ Oracle Database 19c Installation & Configuration Guide

This tutorial provides a **step-by-step guide** to installing and configuring **Oracle Database 19c** on Oracle Linux 8.  
The configuration is aimed at **production-ready setups** with **two independent databases** and **dedicated listeners**.

---

## 📑 Table of Contents
1. [Host Planning](#1-host-planning)
2. [OS Preparation](#2-os-preparation)
3. [User and Directory Setup](#3-user-and-directory-setup)
4. [Oracle Software Installation](#4-oracle-software-installation)
5. [Patch to Supported RU](#5-patch-to-supported-ru)
6. [Listener Configuration](#6-listener-configuration)
7. [Database Creation](#7-database-creation)
8. [Enable ARCHIVELOG & FRA](#8-enable-archivelog--fra)
9. [Systemd Integration](#9-systemd-integration)
10. [Firewall Configuration](#10-firewall-configuration)
11. [Validation](#11-validation)
12. [RMAN Backup Strategy](#12-rman-backup-strategy)
13. [Hardening & Operations Checklist](#13-hardening--operations-checklist)

---

## 1. Host Planning

- **Hostname**: `dbhost1.example.com`  
- **Oracle Base**: `/u01/app/oracle`  
- **Oracle Home**: `/u01/app/oracle/product/19c/dbhome_1`  

### Database Layout
- **DB1 (PROD1)**  
  - Data: `/u02/oradata/PROD1`  
  - FRA: `/u02/fra/PROD1`  

- **DB2 (PROD2)**  
  - Data: `/u03/oradata/PROD2`  
  - FRA: `/u03/fra/PROD2`  

### Listeners
- `LISTENER_PROD1` on TCP `1521`  
- `LISTENER_PROD2` on TCP `15210`  

### Security
- SELinux: Enforcing  
- Firewalld: only TCP 1521 and 15210 open  

---

## 2. OS Preparation

```bash
sudo hostnamectl set-hostname dbhost1.example.com
echo "192.0.2.10  dbhost1.example.com dbhost1" | sudo tee -a /etc/hosts

sudo dnf update -y
sudo dnf install -y oracle-database-preinstall-19c unzip tar
```

---

## 3. User and Directory Setup

```bash
sudo mkdir -p /u01/app/oracle/product/19c/dbhome_1
sudo mkdir -p /u02/oradata/PROD1 /u02/fra/PROD1
sudo mkdir -p /u03/oradata/PROD2 /u03/fra/PROD2
sudo chown -R oracle:oinstall /u01 /u02 /u03
sudo chmod -R 775 /u01 /u02 /u03
```

---

## 4. Oracle Software Installation

Set environment variables for `oracle` user:

```bash
su - oracle

cat <<'EOF' >> ~/.bash_profile
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=/u01/app/oracle/product/19c/dbhome_1
export ORA_INVENTORY=/u01/app/oraInventory
export PATH=$ORACLE_HOME/bin:$PATH
export ORACLE_HOSTNAME=dbhost1.example.com
EOF

source ~/.bash_profile
```

Unzip and run installer:

```bash
su - oracle

cd $ORACLE_HOME
unzip -oq /path/to/LINUX.X64_193000_db_home.zip

export CV_ASSUME_DISTID=OEL7.6

./runInstaller -ignorePrereq \
			   -waitforcompletion \
			   -silent \
               -responseFile $ORACLE_HOME/install/response/db_install.rsp \
               oracle.install.option=INSTALL_DB_SWONLY \
               ORACLE_HOSTNAME=${ORACLE_HOSTNAME} \
               UNIX_GROUP_NAME=oinstall \
               INVENTORY_LOCATION=${ORA_INVENTORY} \
               ORACLE_HOME=${ORACLE_HOME} \
               ORACLE_BASE=${ORACLE_BASE} \
               oracle.install.db.InstallEdition=SE2 \
               oracle.install.db.OSDBA_GROUP=dba \
               oracle.install.db.OSBACKUPDBA_GROUP=dba \
               oracle.install.db.OSDGDBA_GROUP=dba \
               oracle.install.db.OSKMDBA_GROUP=dba \
               oracle.install.db.OSRACDBA_GROUP=dba \
               SECURITY_UPDATES_VIA_MYORACLESUPPORT=false \
               DECLINE_SECURITY_UPDATES=true
```

Run root scripts:

```bash
sudo /u01/app/oraInventory/orainstRoot.sh
sudo $ORACLE_HOME/root.sh
```

---

## 5. Patch to Supported RU

```bash
su - oracle

rm -rf $ORACLE_HOME/OPatch
unzip -oq /path/to/opatch -d $ORACLE_HOME
export PATH=$ORACLE_HOME/OPatch:$PATH

unzip -oq /path/to/db/patch -d /tmp
opatch apply -silent /tmp/patch_dir
```

---

## 6. Listener Configuration

Edit `$ORACLE_HOME/network/admin/listener.ora`:

```ini
ELISTENER_PROD1 =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = dbhost1.example.com)(PORT = 1521))
    )
  )

LISTENER_PROD2 =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = dbhost1.example.com)(PORT = 15210))
    )
  )
```

Edit `$ORACLE_HOME/network/admin/tnsnames.ora`:

```ini
PROD1_LOCAL = (ADDRESS=(PROTOCOL=TCP)(HOST=dbhost1.example.com)(PORT=1521))
PROD2_LOCAL = (ADDRESS=(PROTOCOL=TCP)(HOST=dbhost1.example.com)(PORT=15210))
```

Start listeners:

```bash
lsnrctl start LISTENER_PROD1
lsnrctl start LISTENER_PROD2
```

---

## 7. Database Creation

Example for PROD1:

```bash
su - oracle

dbca -silent -createDatabase \
	 -templateName General_Purpose.dbc \
     -gdbname PROD1 -sid PROD1 \
     -datafileDestination /u02/oradata \
     -characterSet AL32UTF8 \
     -sysPassword 'Zxcv_1234' \
     -systemPassword 'Zxcv_1234' \
     -emConfiguration NONE

export ORACLE_SID=PROD1

sqlplus / as sysdba <<'SQL'
ALTER SYSTEM SET local_listener='PROD1_LOCAL' SCOPE=BOTH;
SHUTDOWN IMMEDIATE;
STARTUP;
SQL

lsnrctl status LISTENER_PROD1
```

Repeat for PROD2 with `/u03/oradata`.

---

## 8. Enable ARCHIVELOG & FRA

```bash
su - oracle

export ORACLE_SID=PROD1

sqlplus / as sysdba <<'SQL'
-- FRA and size
ALTER SYSTEM SET db_recovery_file_dest='/u02/fra/PROD1' SCOPE=BOTH;
ALTER SYSTEM SET db_recovery_file_dest_size=100G SCOPE=BOTH;

-- ARCHIVELOG + FORCE LOGGING
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;
ALTER DATABASE FORCE LOGGING;

-- Verify
ARCHIVE LOG LIST;
SHOW PARAMETER db_recovery_file_dest
SQL
```

Repeat with `/u03/fra/PROD2` for PROD2.

---

## 9. Systemd Integration

Edit `/etc/oratab`:

```ini
PROD1:/u01/app/oracle/product/19c/dbhome_1:Y
PROD2:/u01/app/oracle/product/19c/dbhome_1:Y
```

Create services:

```bash
sudo tee /etc/systemd/system/oracle-listeners.service >/dev/null <<'UNIT'
[Unit]
Description=Oracle Net Listeners
After=network.target

[Service]
User=oracle
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/bash -lc "export
ORACLE_HOME=/u01/app/oracle/product/19c/dbhome_1; export
ORACLE_BASE=/u01/app/oracle; $ORACLE_HOME/bin/lsnrctl start LISTENER_PROD1;
$ORACLE_HOME/bin/lsnrctl start LISTENER_PROD2"
ExecStop=/bin/bash -lc "export
ORACLE_HOME=/u01/app/oracle/product/19c/dbhome_1; export
ORACLE_BASE=/u01/app/oracle; $ORACLE_HOME/bin/lsnrctl stop LISTENER_PROD2;
$ORACLE_HOME/bin/lsnrctl stop LISTENER_PROD1"

[Install]
WantedBy=multi-user.target
UNIT
```
```bash
sudo tee /etc/systemd/system/oracle-databases.service >/dev/null <<'UNIT'
[Unit]
Description=Oracle Databases
After=network.target oracle-listeners.service
Requires=oracle-listeners.service

[Service]
User=oracle
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/bash -lc "export
ORACLE_HOME=/u01/app/oracle/product/19c/dbhome_1; export
ORACLE_BASE=/u01/app/oracle; $ORACLE_HOME/bin/dbstart $ORACLE_HOME"
ExecStop=/bin/bash -lc "export
ORACLE_HOME=/u01/app/oracle/product/19c/dbhome_1; export
ORACLE_BASE=/u01/app/oracle; $ORACLE_HOME/bin/dbshut $ORACLE_HOME"

[Install]
WantedBy=multi-user.target
UNIT
```

Start services:

```bash
sudo systemctl daemon-reload
sudo systemctl enable oracle-listeners.service oracle-databases.service
sudo systemctl start oracle-listeners.service oracle-databases.service
sudo systemctl status oracle-listeners.service oracle-databases.service
```

---

## 10. Firewall Configuration

```bash
sudo firewall-cmd --permanent --add-port=1521/tcp
sudo firewall-cmd --permanent --add-port=15210/tcp
sudo firewall-cmd --reload
```

---

## 11. Validation

```bash
su - oracle

lsnrctl status LISTENER_PROD1
lsnrctl status LISTENER_PROD2
tnsping PROD1_LOCAL
tnsping PROD2_LOCAL
```

---

## 12. RMAN Backup Strategy

```bash
su - oracle

rman target / <<EOF
RUN {
  CONFIGURE CONTROLFILE AUTOBACKUP ON;
  BACKUP INCREMENTAL LEVEL 0 DATABASE PLUS ARCHIVELOG;
  DELETE NOPROMPT OBSOLETE;
}
EOF
```

---

## 13. Hardening & Operations Checklist

- Keep SELinux enforcing  
- Only open required ports  
- Regular RMAN backups  
- Apply quarterly Release Updates

---

## 👤 Author

**Przemyslaw Pradela**  

- ✉️ Email: [przemyslaw.pradela@gmail.com](mailto:przemyslaw.pradela@gmail.com?subject=Oracle%20Database%2019c%20Installation%20%26%20Configuration%20Guide)  

---
