---
name: oracle-ebs-dba
description: |
  Expert Oracle E-Business Suite (EBS) DBA guidance covering R12.1 and R12.2. 
  Use this skill whenever the user mentions Oracle EBS, Oracle Apps, R12, 
  adpatch, adop, AutoConfig, concurrent manager, AD utilities, EBS patching, 
  cloning an EBS instance, FNDLOAD, EBS services (start/stop), apps listener, 
  oacore, oafm, forms, ohs, wls, managed servers, adzdpatch, adutility, 
  fs1/fs2/fs_ne, run edition, patch edition, cutover, hotpatch, 
  EBS tablespaces, APPS schema, applmgr, oracle user, EBS alert log, 
  EBS log files, concurrent requests stuck, concurrent manager down, 
  AutoConfig context file, s_systemname, txkCfgChck, or any task involving 
  administering, patching, cloning, troubleshooting, or maintaining an 
  Oracle EBS environment. Also trigger for questions about EBS-specific 
  SQL queries, AD_BUGS, FND_NODES, FND_CONCURRENT_REQUESTS, 
  FND_CONCURRENT_PROGRAMS, or any fnd_ / ad_ schema objects.
---

# Oracle EBS DBA Skill

Expert knowledge for DBAs administering Oracle E-Business Suite R12.1 and R12.2. 
Always identify which release the user is on before giving patching or service 
management advice — the two releases differ significantly in those areas.

---

## 1. Version Identification

| Feature | R12.1 | R12.2 |
|---|---|---|
| Patch tool | `adpatch` | `adop` |
| Online patching | No | Yes (dual filesystem) |
| Filesystems | Single (APPL_TOP) | fs1, fs2, fs_ne |
| WLS/Fusion Middleware | No | Yes |
| DB requirement | 11gR2 | 12c / 19c |
| adzdpatch | No | Yes (DB-side patching) |

Quick check query to confirm release:
```sql
SELECT RELEASE_NAME FROM FND_PRODUCT_GROUPS;
-- Returns e.g. "12.1.3" or "12.2.10"
```

---

## 2. Environment Layout

### R12.1 Key Paths
```
$APPL_TOP          Applications top (e.g. /u01/oracle/VIS/apps/apps_st/appl)
$ORACLE_HOME       DB Oracle Home
$IAS_ORACLE_HOME   iAS/OHS Oracle Home
$COMMON_TOP        Common top
$INST_TOP          Instance top (config, logs)
$ADMIN_SCRIPTS_HOME  $INST_TOP/admin/scripts
```

### R12.2 Key Paths
```
$EBS_DOMAIN_HOME   WLS domain home
$APPL_TOP          Run edition APPL_TOP  (fs1 or fs2)
$PATCH_BASE        Patch filesystem base
$NE_BASE           Non-editioned filesystem (fs_ne)
$INST_TOP          Instance top
$ADMIN_SCRIPTS_HOME  $INST_TOP/admin/scripts
```

R12.2 dual filesystem — check current run edition:
```bash
cat $NE_BASE/EBSapps.env | grep RUN_BASE
# or
echo $RUN_BASE
```

---

## 3. Starting and Stopping Services

### R12.1 — Start / Stop

```bash
# Source environment first
source /u01/oracle/<SID>/apps/apps_st/appl/APPS<SID>_<hostname>.env

# Stop all services
$ADMIN_SCRIPTS_HOME/adstpall.sh apps/<apps_password>

# Start all services
$ADMIN_SCRIPTS_HOME/adstrtal.sh apps/<apps_password>

# Individual services
$ADMIN_SCRIPTS_HOME/adoacorectl.sh stop   # oacore (Forms/OA)
$ADMIN_SCRIPTS_HOME/adoacorectl.sh start
$ADMIN_SCRIPTS_HOME/adopmnctl.sh stop     # OPMN
$ADMIN_SCRIPTS_HOME/adopmnctl.sh start
$ADMIN_SCRIPTS_HOME/adcmctl.sh stop apps/<apps_password>   # Concurrent Mgr
$ADMIN_SCRIPTS_HOME/adcmctl.sh start apps/<apps_password>

# Apache / OHS
$ADMIN_SCRIPTS_HOME/adapcctl.sh stop
$ADMIN_SCRIPTS_HOME/adapcctl.sh start
```

### R12.2 — Start / Stop

```bash
# Source run edition environment
source $NE_BASE/EBSapps.env run   # or 'patch' for patch edition

# Stop all
$ADMIN_SCRIPTS_HOME/adstpall.sh apps/<apps_password>

# Start all
$ADMIN_SCRIPTS_HOME/adstrtal.sh apps/<apps_password>

# WLS Managed Servers (R12.2 specific)
$ADMIN_SCRIPTS_HOME/adadminsrvctl.sh stop   # Admin server
$ADMIN_SCRIPTS_HOME/adadminsrvctl.sh start
$ADMIN_SCRIPTS_HOME/admanagedsrvctl.sh stop oacore_server1
$ADMIN_SCRIPTS_HOME/admanagedsrvctl.sh start oacore_server1

# OHS
$ADMIN_SCRIPTS_HOME/adoacorectl.sh stop
$ADMIN_SCRIPTS_HOME/adoacorectl.sh start

# Concurrent Manager
$ADMIN_SCRIPTS_HOME/adcmctl.sh stop apps/<apps_password>
$ADMIN_SCRIPTS_HOME/adcmctl.sh start apps/<apps_password>
```

### Check Running Processes (both versions)
```bash
# Check all EBS processes
ps -ef | grep -E "FNDLIBR|FNDSM|oacore|oafm|forms|ohs|wls" | grep -v grep

# Check CM status via SQL
sqlplus apps/<apps_password> <<EOF
SELECT CONCURRENT_QUEUE_NAME, RUNNING_PROCESSES, MAX_PROCESSES, 
       DECODE(CONTROL_CODE,'A','Active','T','Terminate','X','Stop',CONTROL_CODE) STATUS
FROM   FND_CONCURRENT_QUEUES_VL
WHERE  MANAGER_TYPE = '1';
EOF
```

---

## 4. Patching

### R12.1 — adpatch

```bash
# Source env
source $APPL_TOP/APPS<SID>_<hostname>.env

# Run adpatch (interactive)
cd $APPL_TOP
adpatch

# Common adpatch options
adpatch options=nocompiledb,nocompilejsp   # skip DB/JSP compile
adpatch defaultsfile=<file>                # non-interactive
adpatch apply=<patch_dir>/u<patch_num>.drv # apply specific driver

# Restart interrupted patch
adpatch restart=yes
```

**Pre-patch checklist (R12.1):**
1. Backup APPL_TOP and DB (at minimum export of custom objects)
2. Stop all services except DB
3. Check `adpatch` session log: `$APPL_TOP/admin/<SID>/log/`
4. Check for failed workers: `SELECT * FROM AD_DEFERRED_JOBS;`

### R12.2 — adop (Online Patching)

adop uses phases: `prepare → apply → finalize → cutover → cleanup`

```bash
# Source run edition
source $NE_BASE/EBSapps.env run

# Full patch cycle
adop phase=prepare
adop phase=apply patches=<patch_number>    # system stays up during apply
adop phase=finalize
adop phase=cutover                          # brief downtime here
adop phase=cleanup

# Run all phases in one command
adop phase=apply,finalize,cutover,cleanup patches=<patch_number>

# Hotpatch (no downtime, for approved patches only)
adop phase=apply patches=<patch_number> apply_mode=hotpatch

# Apply multiple patches
adop phase=apply patches=<p1>,<p2>,<p3>

# Merge patches first (recommended for multiple)
admrgpch <patch1_dir> <patch2_dir> -merge <merged_dir>
adop phase=apply patches=<merged_patch>

# Resume interrupted adop session
adop phase=apply patches=<patch_number> resume=yes

# Abort failed session (use carefully)
adop phase=abort
```

**R12.2 adop log locations:**
```
$NE_BASE/EBSapps/log/adop_<session_id>/
  apply/             -- apply phase logs
  cutover/           -- cutover logs
  prepare/           -- prepare logs
  adop_<date>.log    -- main session log
```

**Check adop session status:**
```sql
SELECT SESSION_ID, ADOP_SESSION_ID, STATUS, START_DATE, END_DATE
FROM   AD_ADOP_SESSIONS
ORDER  BY START_DATE DESC;
```

**Check patch workers:**
```sql
SELECT WORKER_ID, STATUS, PRODUCT, FILENAME
FROM   AD_WORKERS
WHERE  STATUS NOT IN ('Completed')
ORDER  BY WORKER_ID;
```

---

## 5. AutoConfig

AutoConfig regenerates config files from the context file. Run it after any infrastructure change (hostname, port, IP, profile option changes that affect config).

```bash
# Source environment first

# Run AutoConfig (apps tier)
$ADMIN_SCRIPTS_HOME/adautocfg.sh apps/<apps_password>

# Run AutoConfig (db tier) — as oracle user
$ORACLE_HOME/appsutil/scripts/<CONTEXT_NAME>/adautocfg.sh

# R12.2: run on both run and patch editions
source $NE_BASE/EBSapps.env run
$ADMIN_SCRIPTS_HOME/adautocfg.sh apps/<apps_password>
source $NE_BASE/EBSapps.env patch
$ADMIN_SCRIPTS_HOME/adautocfg.sh apps/<apps_password>
```

**Context file location:**
```bash
# Apps tier
$INST_TOP/appl/admin/<CONTEXT_NAME>.xml   # R12.1
$NE_BASE/inst/apps/<CONTEXT_NAME>/appl/admin/<CONTEXT_NAME>.xml  # R12.2

# DB tier
$ORACLE_HOME/appsutil/<CONTEXT_NAME>.xml
```

**Validate AutoConfig:**
```bash
perl $AD_TOP/bin/admkappsutil.pl   # regenerate appsutil.zip
# Or use txkCfgChck
perl $FND_TOP/patch/115/bin/txkCfgChck.pl \
  -contextfile=<context_file> \
  -outfile=/tmp/cfgchck.html
```

---

## 6. Cloning

### R12.1 Clone (adcfgclone)

**On Source (run as applmgr):**
```bash
# Stage 1: Prepare source
perl $ADMIN_SCRIPTS_HOME/adpreclone.pl dbTier
perl $ADMIN_SCRIPTS_HOME/adpreclone.pl appsTier
```

**Copy files to target** (rsync or OS copy of APPL_TOP, INST_TOP, ORACLE_HOME)

**On Target DB tier (as oracle):**
```bash
perl $ORACLE_HOME/appsutil/clone/bin/adcfgclone.pl dbTier
```

**On Target Apps tier (as applmgr):**
```bash
perl $APPL_TOP/admin/adcfgclone.pl appsTier
```

### R12.2 Clone (adcfgclone — same tool, dual filesystem aware)

```bash
# Source — run preclone on run edition
source $NE_BASE/EBSapps.env run
perl $ADMIN_SCRIPTS_HOME/adpreclone.pl dbTechStack
perl $ADMIN_SCRIPTS_HOME/adpreclone.pl appsTier
perl $ADMIN_SCRIPTS_HOME/adpreclone.pl dbTier

# After copying, target configuration same as R12.1 but
# adcfgclone handles fs1/fs2/fs_ne automatically
perl $APPL_TOP/admin/adcfgclone.pl appsTier
```

**Key clone parameters to update in context file:**
- `s_systemname` — target SID
- `s_hostname` — target hostname
- `s_dbhost` — DB hostname
- `s_apps_jdbc_connect_alias` — JDBC connect string
- `s_appsServerId` — unique server ID

---

## 7. Concurrent Manager Troubleshooting

### Check CM Health
```sql
-- Manager status summary
SELECT Q.CONCURRENT_QUEUE_NAME,
       Q.USER_CONCURRENT_QUEUE_NAME,
       Q.RUNNING_PROCESSES,
       Q.MAX_PROCESSES,
       Q.CONTROL_CODE,
       P.OS_PROCESS_ID
FROM   FND_CONCURRENT_QUEUES   Q,
       FND_CONCURRENT_PROCESSES P
WHERE  Q.CONCURRENT_QUEUE_ID = P.CONCURRENT_QUEUE_ID(+)
AND    Q.MANAGER_TYPE = '1'
ORDER  BY Q.CONCURRENT_QUEUE_NAME;

-- Stuck/long-running requests
SELECT REQUEST_ID, REQUESTED_BY, PHASE_CODE, STATUS_CODE,
       ACTUAL_START_DATE, CONCURRENT_PROGRAM_NAME,
       ROUND((SYSDATE - ACTUAL_START_DATE)*24*60,1) RUNNING_MINS
FROM   FND_CONCURRENT_REQUESTS FCR,
       FND_CONCURRENT_PROGRAMS FCP
WHERE  FCR.CONCURRENT_PROGRAM_ID = FCP.CONCURRENT_PROGRAM_ID
AND    FCR.PHASE_CODE = 'R'
ORDER  BY ACTUAL_START_DATE;

-- Pending requests older than 30 minutes
SELECT REQUEST_ID, REQUESTED_BY, CONCURRENT_PROGRAM_NAME,
       REQUESTED_START_DATE,
       ROUND((SYSDATE - REQUESTED_START_DATE)*24*60,1) WAIT_MINS
FROM   FND_CONCURRENT_REQUESTS FCR,
       FND_CONCURRENT_PROGRAMS FCP
WHERE  FCR.CONCURRENT_PROGRAM_ID = FCP.CONCURRENT_PROGRAM_ID
AND    FCR.PHASE_CODE = 'P'
AND    FCR.STATUS_CODE = 'I'
AND    REQUESTED_START_DATE < SYSDATE - 30/1440
ORDER  BY REQUESTED_START_DATE;
```

### Kill a Stuck Request
```sql
-- Mark request as cancelled (safe, CM cleans up)
UPDATE FND_CONCURRENT_REQUESTS
SET    STATUS_CODE = 'X', PHASE_CODE = 'C'
WHERE  REQUEST_ID = <request_id>
AND    PHASE_CODE = 'R';
COMMIT;

-- If OS process needs killing (get OS PID first)
SELECT OS_PROCESS_ID FROM FND_CONCURRENT_PROCESSES
WHERE  CONCURRENT_PROCESS_ID = (
  SELECT CONTROLLING_MANAGER FROM FND_CONCURRENT_REQUESTS
  WHERE  REQUEST_ID = <request_id>);
-- Then: kill -9 <OS_PID> from OS
```

### Restart CM Cleanly
```bash
# Stop
$ADMIN_SCRIPTS_HOME/adcmctl.sh stop apps/<apps_password>

# Verify no FNDLIBR/FNDSM processes remain
ps -ef | grep -E "FNDLIBR|FNDSM" | grep -v grep

# Clean up ICM row if CM thinks it's still running
sqlplus apps/<apps_password> <<EOF
UPDATE FND_CONCURRENT_QUEUES
SET    RUNNING_PROCESSES = 0
WHERE  CONCURRENT_QUEUE_NAME = 'STANDARD';
COMMIT;
EOF

# Start
$ADMIN_SCRIPTS_HOME/adcmctl.sh start apps/<apps_password>
```

---

## 8. Log File Locations

### Application Tier Logs
```bash
# Concurrent request logs and out files
$APPLCSF/log/<request_id>.log
$APPLCSF/out/<request_id>.out
# or via SQL:
SELECT LOGFILE_NAME, OUTFILE_NAME FROM FND_CONCURRENT_REQUESTS
WHERE REQUEST_ID = <id>;

# AutoConfig log
$INST_TOP/admin/log/

# adpatch log (R12.1)
$APPL_TOP/admin/<SID>/log/adpatch_<date>.log

# adop log (R12.2)
$NE_BASE/EBSapps/log/adop_<session>/

# Apache/OHS error log
$INST_TOP/logs/ora/10.1.3/Apache/error_log    # R12.1
$INST_TOP/logs/                                # R12.2 — check subdirs

# WLS logs (R12.2)
$EBS_DOMAIN_HOME/servers/oacore_server1/logs/
$EBS_DOMAIN_HOME/servers/oafm_server1/logs/
$EBS_DOMAIN_HOME/servers/forms_server1/logs/
```

### Database Tier Logs
```bash
# Alert log
$ORACLE_BASE/diag/rdbms/<db_name>/<SID>/trace/alert_<SID>.log
# Pre-11g:
$ORACLE_HOME/admin/<SID>/bdump/alert_<SID>.log

# Listener log
$ORACLE_BASE/diag/tnslsnr/<hostname>/listener/alert/log.xml
```

---

## 9. Key EBS DBA SQL Queries

### Patch Level & Installation
```sql
-- Applied patches
SELECT PATCH_NAME, PATCH_TYPE, CREATION_DATE
FROM   AD_BUGS
WHERE  BUG_NUMBER = '&patch_number';

-- All applied patches (recent first)
SELECT BUG_NUMBER, CREATION_DATE, LAST_UPDATE_DATE
FROM   AD_BUGS
ORDER  BY CREATION_DATE DESC
FETCH FIRST 50 ROWS ONLY;

-- Product versions
SELECT APPLICATION_SHORT_NAME, PATCH_LEVEL
FROM   FND_PRODUCT_INSTALLATIONS
ORDER  BY APPLICATION_SHORT_NAME;

-- AD patch history
SELECT PATCH_NAME, PATCH_TYPE, SOURCE_CODE, CREATION_DATE
FROM   AD_APPLIED_PATCHES
ORDER  BY CREATION_DATE DESC;
```

### Users & Responsibilities
```sql
-- Active users
SELECT USER_NAME, DESCRIPTION, START_DATE, END_DATE, LAST_LOGON_DATE
FROM   FND_USER
WHERE  (END_DATE IS NULL OR END_DATE > SYSDATE)
ORDER  BY LAST_LOGON_DATE DESC NULLS LAST;

-- User responsibilities
SELECT FU.USER_NAME, FR.RESPONSIBILITY_NAME, FUR.START_DATE, FUR.END_DATE
FROM   FND_USER_RESP_GROUPS_DIRECT FUR,
       FND_USER                    FU,
       FND_RESPONSIBILITY_TL       FR
WHERE  FUR.USER_ID           = FU.USER_ID
AND    FUR.RESPONSIBILITY_ID = FR.RESPONSIBILITY_ID
AND    FU.USER_NAME          = '&username'
AND    FR.LANGUAGE           = USERENV('LANG');
```

### System / Nodes
```sql
-- Registered nodes
SELECT NODE_NAME, SERVER_ID, SERVER_ADDRESS, SUPPORT_FORMS,
       SUPPORT_WEB, SUPPORT_BATCH, SUPPORT_ADMIN, STATUS
FROM   FND_NODES;

-- Profile option values
SELECT FP.PROFILE_OPTION_NAME,
       FPT.USER_PROFILE_OPTION_NAME,
       DECODE(FPV.LEVEL_ID,
              10001,'Site', 10002,'App', 10003,'Resp', 10004,'User') LEVEL_SET,
       FPV.PROFILE_OPTION_VALUE
FROM   FND_PROFILE_OPTIONS     FP,
       FND_PROFILE_OPTIONS_TL  FPT,
       FND_PROFILE_OPTION_VALUES FPV
WHERE  FP.PROFILE_OPTION_ID   = FPT.PROFILE_OPTION_ID
AND    FP.PROFILE_OPTION_ID   = FPV.PROFILE_OPTION_ID
AND    FPT.LANGUAGE           = USERENV('LANG')
AND    UPPER(FP.PROFILE_OPTION_NAME) LIKE UPPER('%&search%')
ORDER  BY FP.PROFILE_OPTION_NAME, FPV.LEVEL_ID;
```

---

## 10. Tablespace Management

```sql
-- Tablespace usage
SELECT DF.TABLESPACE_NAME,
       ROUND(DF.TOTAL_MB,1)                            TOTAL_MB,
       ROUND(DF.TOTAL_MB - NVL(FS.FREE_MB,0), 1)      USED_MB,
       ROUND(NVL(FS.FREE_MB,0), 1)                     FREE_MB,
       ROUND((1 - NVL(FS.FREE_MB,0)/DF.TOTAL_MB)*100,1) PCT_USED
FROM  (SELECT TABLESPACE_NAME, SUM(BYTES)/1048576 TOTAL_MB
       FROM   DBA_DATA_FILES GROUP BY TABLESPACE_NAME)   DF,
      (SELECT TABLESPACE_NAME, SUM(BYTES)/1048576 FREE_MB
       FROM   DBA_FREE_SPACE  GROUP BY TABLESPACE_NAME)  FS
WHERE  DF.TABLESPACE_NAME = FS.TABLESPACE_NAME(+)
ORDER  BY PCT_USED DESC;

-- Add datafile to tablespace
ALTER TABLESPACE APPS_TS_TX_DATA
ADD DATAFILE '/u01/oradata/<SID>/apps_ts_tx_data02.dbf'
SIZE 10G AUTOEXTEND ON NEXT 512M MAXSIZE 30G;

-- Resize existing datafile
ALTER DATABASE DATAFILE '/u01/oradata/<SID>/apps_ts_tx_data01.dbf'
RESIZE 20G;
```

---

## 11. Common Issues & Quick Fixes

### "Could not connect to ICM"
1. Check if ICM process is running: `ps -ef | grep FNDLIBR | grep -v grep`
2. Check `$ADMIN_SCRIPTS_HOME/adcmctl.sh status apps/<pw>`
3. Check DB connectivity from apps tier: `tnsping <SID>`
4. Review ICM log: look in `$APPLCSF/log/` for most recent CM log

### Forms / OAF Not Loading
```bash
# R12.1 — check oacore
$ADMIN_SCRIPTS_HOME/adoacorectl.sh status
# Bounce if needed
$ADMIN_SCRIPTS_HOME/adoacorectl.sh stop && $ADMIN_SCRIPTS_HOME/adoacorectl.sh start

# R12.2 — check WLS managed servers
$ADMIN_SCRIPTS_HOME/admanagedsrvctl.sh status oacore_server1
# If FAILED, restart
$ADMIN_SCRIPTS_HOME/admanagedsrvctl.sh stop  oacore_server1
$ADMIN_SCRIPTS_HOME/admanagedsrvctl.sh start oacore_server1
```

### AutoConfig Fails
1. Check `$INST_TOP/admin/log/` for latest AutoConfig log — look for `ERROR` lines
2. Validate context file XML is well-formed: `xmllint --noout <context_file.xml>`
3. Ensure `appsutil.zip` is current: `perl $AD_TOP/bin/admkappsutil.pl`
4. Check DB tier AutoConfig ran first (apps tier depends on it for JDBC URLs)

### R12.2: adop prepare Fails
```bash
# Check fs synchronization
adop phase=prepare
# Look in adop log for "FAILED" — common causes:
# - Patch edition filesystem out of sync (run adop cleanup first)
# - WLS admin server down
# - Invalid session in AD_ADOP_SESSIONS

# Force cleanup of incomplete session
adop phase=cleanup mode=full
```

### Login Page Not Accessible
```bash
# Check listener
lsnrctl status

# Check OHS
$ADMIN_SCRIPTS_HOME/adapcctl.sh status   # R12.1
# R12.2 — check OHS via OPMN or directly:
ps -ef | grep httpd | grep -v grep

# Check apps context — port correct?
grep s_webport $INST_TOP/appl/admin/*.xml
```

---

## 12. Security & User Management

```sql
-- Unlock FND user
BEGIN
  FND_USER_PKG.UpdateUser(
    x_user_name      => 'USERNAME',
    x_owner          => 'CUST',
    x_end_date       => FND_API.G_MISS_DATE,
    x_password_lifespan_days => FND_API.G_MISS_NUM
  );
  COMMIT;
END;
/

-- Reset FND user password
BEGIN
  FND_USER_PKG.ChangePassword(
    username    => 'USERNAME',
    newpassword => 'Welcome1'
  );
  COMMIT;
END;
/

-- Expire all user passwords (force reset on next login)
UPDATE FND_USER
SET    PASSWORD_DATE = NULL
WHERE  (END_DATE IS NULL OR END_DATE > SYSDATE);
COMMIT;
```

---

## Reference Files

For deeper guidance, refer to:
- `references/patching-scenarios.md` — common patching scenarios and edge cases
- `references/cloning-checklist.md` — pre/post clone checklist
- `references/cm-troubleshooting.md` — extended concurrent manager diagnostics
