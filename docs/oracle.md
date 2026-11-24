# Oracle Database Cheat Sheet

> Use `sqlplus "$DB_CONNECT"` where `DB_CONNECT` is an environment variable. **Never** type `user/password` inline: it ends up in shell history and `ps`.

## Restore points (safe rollback around a release)
Requirements: connect `AS SYSDBA`/`SYSBACKUP`, database in `ARCHIVELOG` mode.
```sql
ARCHIVE LOG LIST;
SELECT log_mode, flashback_on FROM v$database;
ALTER DATABASE FLASHBACK ON;

CREATE RESTORE POINT RESTORE_20250825 GUARANTEE FLASHBACK DATABASE;
SELECT name, guarantee_flashback_database, time FROM v$restore_point ORDER BY time DESC;

-- Roll back (database must be mounted)
FLASHBACK DATABASE TO RESTORE POINT RESTORE_20250825;

-- Always drop it after the release is accepted, otherwise the fast recovery area fills up
DROP RESTORE POINT RESTORE_20250825;
```
Scripted from a shell (`OS auth`, no password on the command line):
```bash
sqlplus -S / as sysdba <<SQL
set markup csv on
select log_mode from v\$database;
create restore point ${ENV_NAME}_${DATE} guarantee flashback database;
select time, restore_point_time, preserved, name from v\$restore_point order by time desc;
set markup csv off
SQL
```
If the archive area fills up, remove old archive logs (RMAN) before anything else.

## Health queries
```sql
-- Invalid objects (add to pre/post install checks)
SELECT COUNT(*) FROM dba_objects WHERE status <> 'VALID';
SELECT owner, object_type, object_name
  FROM dba_objects WHERE status <> 'VALID'
 ORDER BY owner, object_type;

-- Clock vs OS
SELECT SYSDATE FROM DUAL;
SELECT DBTIMEZONE FROM DUAL;
```
```bash
lsnrctl start
lsnrctl start $ORACLE_SID
ps -fea | egrep "pmon|inh"      # instance and listener processes
```

## Scripting output
```sql
SET MARKUP CSV ON
SELECT operator_login_name, operator_full_name FROM operator WHERE operator_login_name = '<ID>';
SET MARKUP CSV OFF
```
Clean batch output:
```sql
SET PAGESIZE 0
SET FEEDBACK OFF
SET LINESIZE 32000
SET TAB OFF
SET TIMING OFF
```
Run long scripts detached:
```bash
nohup sqlplus -S -L "$DB_CONNECT" @/path/to/script.sql &
```
Make a script continue after errors: replace `whenever sqlerror exit sql.sqlcode;` with `whenever sqlerror continue;`.

## Table housekeeping
```sql
-- Copy structure only
CREATE TABLE ZCHARGE_COPY AS SELECT * FROM CHARGE WHERE rownum < 1;

-- Generate rename statements
SELECT 'rename ' || table_name || ' to Z' || table_name || ';'
  FROM user_tables WHERE table_name LIKE 'APP%';

TRUNCATE TABLE <TABLE>;
COMMIT;
```

## Rotating an application DB user's password
1. `ALTER USER "<APP_USER>" IDENTIFIED BY "<NEW_PASSWORD>";` (take the value from your vault)
2. Update every interface definition that stores it (Admin GUI > Maintenance > Server > Interface Definition > Connection Details).
3. Regenerate interfaces and restart the affected servers:
```bash
generate_interfaces -w 'INTERFACE_DEFN_ID=<ID>'
svrestart -bmp <APP>
```
Expect a short outage and disconnected sessions on the instances that host the app.

## Statistics
```bash
dbanalyze -sample 100% child 32
```
