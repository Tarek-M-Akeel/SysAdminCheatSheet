# Tarek Linux and Database Command Cheat Sheet

- [General Linux Commands](#general-linux-commands)
   * [File Operations and Comparisons in Bash](#file-operations-and-comparisons-in-bash)
      + [Comparisons:](#comparisons)
      + [File Operations:](#file-operations)
   * [Date Formatting](#date-formatting)
   * [Network Utilities](#network-utilities)
   * [File Management](#file-management)
   * [System Utilities](#system-utilities)
   * [File Comparison](#file-comparison)
   * [User Management](#user-management)
   * [LVM Management](#lvm-management)
   * [Dotnet Build Commands](#dotnet-build-commands)
   * [Text Processing](#text-processing)
- [Database-Related Commands](#database-related-commands)
   * [PostgreSQL](#postgresql)
   * [MySQL/MariaDB](#mysqlmariadb)
   * [Database Dump Manipulation](#database-dump-manipulation)
   * [Replica Status Check Script](#replica-status-check-script)
   * [MariaDB Slave Optimization](#mariadb-slave-optimization)
   * [Database Recovery Tips](#database-recovery-tips)
- [Oracle Database Commands](#oracle-database-commands)
   * [Listener Control](#listener-control)
   * [Data Pump Directory](#data-pump-directory)
   * [Export/Import Operations (Data Pump)](#exportimport-operations-data-pump)
      + [Schema-level Export](#schema-level-export)
      + [Full Database Export](#full-database-export)
      + [Schema-level Import](#schema-level-import)
      + [Full Database Import](#full-database-import)
      + [Schema-specific Import with Remote Connection](#schema-specific-import-with-remote-connection)
   * [Traditional Export/Import (exp/imp)](#traditional-exportimport-expimp)
   * [User Management](#user-management-1)
      + [Create User with Privileges](#create-user-with-privileges)
   * [Database Administration](#database-administration)
      + [Find Backup Directory](#find-backup-directory)
      + [Drop Schema](#drop-schema)
      + [Kill Sessions](#kill-sessions)
      + [Rename Schema](#rename-schema)
      + [Clean Up Non-Oracle Schemas](#clean-up-non-oracle-schemas)
      + [Password Change](#password-change)
      + [Create Directory/Tablespace](#create-directorytablespace)
- [LVM Commands](#lvm-commands)
- [Wildcard Certificate Update Procedure](#wildcard-certificate-update-procedure)

## General Linux Commands

### File Operations and Comparisons in Bash
```bash
if [ -FileOperations x -Comparisons y ]; then
```

#### Comparisons:
- `-eq` - equal to
- `-ne` - not equal to
- `-lt` - less than
- `-le` - less than or equal to
- `-gt` - greater than
- `-ge` - greater than or equal to

#### File Operations:
- `-s` - file exists and is not empty
- `-f` - file exists and is not a directory
- `-d` - directory exists
- `-x` - file is executable
- `-w` - file is writable
- `-r` - file is readable

### Date Formatting
```bash
date
%T   # time; same as %H:%M:%S
%F   # full date; same as %Y-%m-%d
%Y   # year
%m   # month (01..12)
%d   # day of month (e.g., 01)
date +%F--%T
date +%Y-%m-%d--%H:%M:%S
```

### Network Utilities
```bash
curl https://icanhazip.com/  # Get public IP
sudo date -s "$(curl -s --head http://google.com | grep ^Date: | sed 's/Date: //g')"  # Time sync
```

### File Management
```bash
# Find and remove files older than 7 days
find /path/to/ -type f -mtime +7 -name '*.gz' -execdir rm -- '{}' \;
find /path/to/ -type f -mtime +7 -name '*.gz' -delete;

# Find large files
find / -xdev -size +1G -printf "%k  %p\n" | sort -n

# Remove Windows line endings
sed -i 's/\r//g' filename
```

### System Utilities
```bash
# Package installation
dnf install curl wget net-tools bind-utils bash-completion screen tcpdump sysstat htop ncdu certbot telnet tar unar iotop vim rsync iftop lsof jq netcat unzip

# Journal logs
journalctl -S "-30m or -1h" -xeu service

# DOS attack detection
netstat -ant | grep EST | grep 80 | awk '{print $5}' | sed -e "s/::ffff://" | awk -F: '{print $1}' | sort | uniq -c

# SSL certificate check
openssl s_client -connect mail.example.com:465 -servername mail.example.com 2>/dev/null | openssl x509 -noout -dates
```

### File Comparison
```bash
cmp /path/to/file1 /path/to/file2
```

### User Management
```bash
usermod -L -e 1 username  # Lock user and expire account
```

### LVM Management

See the [LVM Commands](#lvm-commands) section below for detailed LVM operations.


### Dotnet Build Commands
```bash
for i in $(find . -name "appsettings.json" | awk -F'/' '{print $2}'); do 
  dotnet publish -c 'Release' /p:ErrorOnDuplicatePublishOutputFiles=false -o /usr/local/tatweer/$i $i
done

for i in $(ls | grep -v sln); do 
  /usr/bin/dotnet publish /p:ErrorOnDuplicatePublishOutputFiles=false -c 'Release' -o publish_$i/ $i
done
```

### Text Processing
```bash
cat file | grep -v "#"  # Exclude lines with #
```

## Database-Related Commands

### PostgreSQL
```sql
DROP DATABASE mydb WITH (FORCE);  # Force drop database
```

### MySQL/MariaDB
```sql
-- Replication commands
STOP SLAVE;
SET GLOBAL sql_slave_skip_counter = 1;
START SLAVE;

-- Database dump
mysqldump --routines --skip-lock-tables --master-data --databases DB1 DB2 | gzip -c | ssh user@host "cat > ~/replicarestore_$(date +%Y%m%d).sql.gz"

-- List databases for dropping (excluding system DBs)
mysql -u admin -p$(cat /etc/psa/.psa.shadow) -e "SELECT CONCAT('DROP DATABASE ', SCHEMA_NAME, ';') FROM information_schema.SCHEMATA WHERE SCHEMA_NAME NOT IN ('information_schema', 'performance_schema', 'mysql', 'sys')"

-- Database check and repair
mysqlcheck -u admin -p$(cat /etc/psa/.psa.shadow) -A --auto-repair
```

### Database Dump Manipulation
```bash
# Remove specific database from dump
sed '/^-- Current Database: `mysql`/,/^-- Current Database: `/d' Fulldump.sql > CleanDump.sql

# Extract specific database from dump
sed -n '/^-- Current Database: `dbname`/,/^-- Current Database: `/p' alldatabases.sql > output.sql
```

### Replica Status Check Script
```bash
#!/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

IOSLAVEALIVE="$(mysql -u check -p'check-slave' -e "SHOW SLAVE STATUS \G" | grep Slave_IO_Running | gawk -F: '{ print $2 }')"
SQLSLAVEALIVE="$(mysql -u check -p'check-slave' -e "SHOW SLAVE STATUS \G" | grep Slave_SQL_Running | gawk -F: '{ print $2 }')"
TIMEBEHINDMASTER="$(mysql -u check -p'check-slave' -e "SHOW SLAVE STATUS \G" | grep Seconds_Behind_Master | gawk -F: '{ print $2 }')"

if [ $IOSLAVEALIVE = "No" -o $SQLSLAVEALIVE = "No" -o $TIMEBEHINDMASTER -ge 7200 ]; then
  mail -s 'Replication Alert' admin@example.com <<< 'Please check database replication status.'
fi
```

### MariaDB Slave Optimization
```ini
innodb_buffer_pool_size=1G
innodb_flush_log_at_trx_commit=2
innodb_flush_method=O_DIRECT
sync_binlog=0
slave_compressed_protocol=1
```

### Database Recovery Tips

1. Set `innodb_force_recovery` to 1 and start MariaDB
2. If that fails, try value 2
3. If successful, dump databases with `mariadb-dump`
4. Verify tables with `mariadb-check --all-databases`
5. Drop and recreate corrupted databases
6. Remove ib* files from data directory
7. Restore from backups/dumps


## Oracle Database Commands

### Listener Control
```bash
lsnrctl status
```

### Data Pump Directory
```sql
DATA_PUMP_DIR = /u01/app/oracle/admin/orcl/dpdump/
```

### Export/Import Operations (Data Pump)

#### Schema-level Export
```bash
expdp \'/ as sysdba\' directory=DATA_PUMP_DIR dumpfile=dump-$(date +%Y_%m_%d).dmp logfile=dump-$(date +%Y_%m_%d).log schemas=x,y

```
#### Full Database Export
```bash
expdp \'/ as sysdba\' full=Y directory=backup dumpfile=dmp-$(date +%Y_%m_%d).dmp logfile=dmp-$(date +%Y_%m_%d).log

expdp \'sys/Password@localhost:1521/orcl as sysdba\' full=Y directory=backup dumpfile=dump_$(date +%Y_%m_%d-%s).dmp logfile=dump$(date +%Y_%m_%d-%s).log
```

#### Schema-level Import
```bash
impdp \'/ as sysdba\' directory=DATA_PUMP_DIR dumpfile=schema-dump.dmp logfile=schema-dump_$(date +%Y_%m_%d-%s).log schemas=schema1
```

#### Full Database Import
```bash
impdp \'/ as sysdba\' full=Y directory=backup dumpfile=FULL_EXPDP_DB_THU.DMP logfile=FULL_impdp_DB_THU.DMP-$(date +%Y_%m_%d).log

```

#### Schema-specific Import with Remote Connection
```bash
impdp \'sys/password@192.168.1.1:1521/orcl as sysdba\' full=Y directory=backup dumpfile=FULL_EXPDP_DB_THU.dmp logfile=FULL_impdp_DB_THU-$(date +%Y_%m_%d-%s).log
```

### Traditional Export/Import (exp/imp)
```bash
imp \'/ as sysdba\' fromuser=schema1 touser=schema2 file=/u01/app/oracle/admin/orcl/dpdump/dump-ex.dmp log=/u01/app/oracle/admin/orcl/dpdump/dump-ex.log

exp \'/ as sysdba\' owner=schema1,schema2 file=/home/oracle/schema1-schema2-$(date +%Y_%m_%d).dmp log=/home/oracle/schema1-schema2-$(date +%Y_%m_%d).log
```

### User Management

#### Create User with Privileges
```sql
CREATE USER User IDENTIFIED BY User
  DEFAULT TABLESPACE User_OPR
  TEMPORARY TABLESPACE TEMP
QUOTA UNLIMITED ON User_OPR_IDX
QUOTA UNLIMITED ON User_SCD
QUOTA UNLIMITED ON User_OPR
QUOTA UNLIMITED ON User_SCD_IDX
QUOTA UNLIMITED ON User_DCD
QUOTA UNLIMITED ON User_DCD_IDX;

GRANT CREATE PUBLIC SYNONYM TO User;
GRANT CREATE ANY SEQUENCE TO User;
GRANT DROP PUBLIC SYNONYM TO User;
GRANT CREATE ANY SNAPSHOT TO User;
GRANT DROP ANY SNAPSHOT TO User;
GRANT DROP ANY SEQUENCE TO User;
GRANT UNLIMITED TABLESPACE TO User;
GRANT CONNECT TO User;
GRANT RESOURCE TO User;
GRANT DBA TO User;
GRANT SELECT ANY DICTIONARY TO User;

-- For 12c:
GRANT EXECUTE ON sys.dbms_crypto TO User;

-- Additional grants:
GRANT DROP ANY MATERIALIZED VIEW TO User;
GRANT CREATE ANY MATERIALIZED VIEW TO User;

ALTER USER User QUOTA UNLIMITED ON User_spr;
ALTER USER User QUOTA UNLIMITED ON User_spr_idx;
```

### Database Administration

#### Find Backup Directory
```sql
SELECT DIRECTORY_NAME, DIRECTORY_PATH FROM dba_directories WHERE DIRECTORY_NAME = 'name_folder';
```

#### Drop Schema
```sql
DROP USER User_TST CASCADE;
```

#### Kill Sessions
```sql
-- Generate kill statements for specific users
SELECT 'alter system kill session ''' || sid || ',' || serial# || ''' immediate;' stmts 
FROM v$session WHERE username = 'User_TST';

-- Execute kill session
ALTER SYSTEM KILL SESSION '38,50144';
```

#### Rename Schema
```sql
SELECT user#,NAME FROM SYS.user$ WHERE NAME='usertest';
UPDATE USER$ SET NAME='realuser' WHERE USER#=154;
```

#### Clean Up Non-Oracle Schemas
```sql
SET SERVEROUTPUT ON
BEGIN
    FOR c IN (SELECT DISTINCT username FROM dba_users WHERE ORACLE_MAINTAINED='N') LOOP
    DBMS_OUTPUT.PUT_LINE(c.username);
    EXECUTE IMMEDIATE ('DROP USER "'||c.username||'" CASCADE');
    END LOOP;
END;
/
```

#### Password Change
```sql
ALTER USER SYS IDENTIFIED BY [password];
```

#### Create Directory/Tablespace
```sql
CREATE DIRECTORY backup AS '/u01/app/oracle/backup/';

CREATE TABLESPACE EXAMPLE DATAFILE 'EXAMPLE.dbf' SIZE 100m AUTOEXTEND ON;
CREATE TABLESPACE REPMAN_IDX DATAFILE 'REPMAN_IDX.dbf' SIZE 100m AUTOEXTEND ON;
```
## LVM Commands

```bash
# Check disk space
df -h
lsblk

# Rescan disks
echo "- - -" | tee /sys/class/scsi_host/host*/scan

# Rescan disk size
echo "1" > /sys/class/block/sd{letter}/device/rescan

# Partition creation with parted
parted /dev/sdb
mklabel gpt
mkpart primary ext4 0% 100%
print
quit

# Update partition table
partprobe

# LVM operations
pvcreate /dev/sdb1
vgdisplay
vgextend root-vg /dev/sdb1
lvdisplay
lvextend -l +100%FREE /dev/root-vg/root-lv

# Filesystem resize
resize2fs /dev/root-vg/root-lv  # For ext4
xfs_growfs /dev/root-vg/root-lv # For XFS
```

## Wildcard Certificate Update Procedure

```bash
cd /path/to/certs/
mkdir old.$(date +%Y%m%d)
mv *.pem old.$(date +%Y%m%d)/
tar xzf *.tar.gz --strip-components 1
rm *.tar.gz
rename # 1 *.pem  # Replace # with actual number
dn -s reload
```
