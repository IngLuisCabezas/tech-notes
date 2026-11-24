# HA Cluster and Application Operations Runbook

> Operational notes for a multi-instance, Pacemaker/Corosync-backed enterprise billing platform running on Tuxedo and Oracle.
> Vendor tool names (`sv_*`, `ecp`, `cfg`, `da_dump`, ...) are kept as-is. Environment names, hosts and users are replaced by placeholders.

| Placeholder | Meaning |
|---|---|
| `$ATA_HOME` | Application home directory of the environment owner |
| `<ENV>` | Environment name (dev, sit, uat, stg, prd) |
| `SV1..SVn` | Application instances (SV1 usually hosts the active PE) |

## Contents
1. [Status and validation](#1-status-and-validation)
2. [Clean stop and start](#2-clean-stop-and-start)
3. [HA cluster (Pacemaker / Corosync)](#3-ha-cluster-pacemaker--corosync)
4. [Rolling a node out of service](#4-rolling-a-node-out-of-service)
5. [PE restricted mode and logins](#5-pe-restricted-mode-and-logins)
6. [Logs: where to look](#6-logs-where-to-look)
7. [Tuxedo and process recovery](#7-tuxedo-and-process-recovery)
8. [Adapters (REST / SOAP / Diameter)](#8-adapters-rest--soap--diameter)
9. [Turning traces off](#9-turning-traces-off)
10. [Installation and release](#10-installation-and-release)
11. [Config export / import](#11-config-export--import)
12. [Maintenance window: disable and re-enable DCEA](#12-maintenance-window-disable-and-re-enable-dcea)
13. [Pre / post install checklist](#13-pre--post-install-checklist)
14. [Reporting server (MySQL + Apache)](#14-reporting-server-mysql--apache)

---

## 1. Status and validation
```bash
sv_status                 # overall state
sv_status -cluster
svstatus
for i in {1..12}; do sv_status SV$i; done

da_dump InstanceStatus
da_dump CustomerPartition
da_dump ClusterStandby

tuxusers -r               # logged-in client users (web)
tuxusers -b               # busy users
pe_admin -u               # logged-in PE users
w                         # unix users
relinfo                   # installed version
```

Typical error lines and what they mean:
```
<E90000> ERROR: SFM:52 is 'Dead' when it is supposed to be 'Started'
<E90000> ERROR: RSP1100.5 (f_rsp) is 'INACTIVE' when expected to be 'ACTIVE'
<W90001> Application is partially running
```

## 2. Clean stop and start
```bash
# Escalating stop: graceful -> kill -> kill -kill
sv_stop || sv_stop -kill || sv_stop -kill -kill
sv_stop || sv_stop -k -k; ipcrm -a
sv_stop -cluster || sv_stop -cluster -kill
{ sv_stop -cluster || sv_stop -cluster -kill; } || true

# Last resort: kills your own session too, releases memory and processes
pkill -9 -u <ENV_USER>

ipcs -a                   # shared memory / semaphores still allocated?
ipcrm -a                  # remove them
pu                        # any perl or sv commands still running? kill -9 them

sv_start -nocb -pe -parallel 36
sv_start -nocb -nope
sv_start -cluster
```

Deep clean before a cold start:
```bash
sv_stop -cluster
billing_clean "$(whoami)" -r -F
lsof | egrep "deleted|COMMAND"
ipcrm -a; ipcs -a
sv_start -cluster
```

Broadcast a message to logged-in users before a restart:
```bash
ecp "biWall&('Instance <SVn> is being shut down for a clean restart in 1 minute. Please save your work and log out.')"
```

## 3. HA cluster (Pacemaker / Corosync)
```bash
systemctl status corosync pacemaker
pcs status
pcs status corosync
pcs cluster status
pcs property show maintenance-mode
pcs property set maintenance-mode=false
pcs constraint show resources
pcs resource failures
pcs resource cleanup
pcs cluster start --all
pcs property set stonith-enabled=false      # lab/test only
```

Application-level wrappers:
```bash
sv_ha_ctl cluster_status
sv_ha_ctl cluster_start
sv_ha_ctl cluster_stop
sv_ha_ctl instance_disable SVx
sv_ha_ctl instance_enable  SVx
sv_ha_ctl instance_restore SVx
sv_ha_ctl resource_clean pe -allinst
sv_taf_ctl status
```

**Practical recovery sequence**
```bash
systemctl status corosync pacemaker
pcs cluster start --all          # on every node
pcs cluster status
pcs resource cleanup
sv_ha_ctl cluster_start
sv_ha_ctl instance_restore SVx
sv_ha_ctl cluster_status
```

## 4. Rolling a node out of service
```bash
# 1. Pull SV3 out of the cluster
sv_ha_ctl instance_disable SV3
sv_stop -nocb -pe

# 2. Do the work ...

# 3. Bring it back
sv_start -nocb -pe -parallel 36
sv_ha_ctl instance_restore SV3

# 4. Verify (from SV1)
sv_ha_ctl cluster_status
sv_status -cluster
svstatus
```
Hub-only move back (example SV5 -> SV4): verify PE is `STANDBY` on SV4, then `instance_disable SV5` and `instance_restore SV5`.

## 5. PE restricted mode and logins
```bash
pe_admin -rc                    # complete a PE start that came up RESTRICTED
pe_admin -ks '<user>'           # kill a user's session
pe_debug -r

# Controlled lock-out for maintenance
c_sap -k NOLOGIN                # 1) block new logins
c_sap -k ROLLBACK               # 2) roll back and log out non-admin sessions
c_sap -k SHUTDOWN               # 3) optional: log out admins and shut PE down
c_sap -k RESTORE                # 4) back to normal
```
After `NOLOGIN`, `sv_status` shows: *"PE Server is RUNNING - SVx Active - No new logins allowed (except for admin users)"*.

PE Errors:
```bash
grep -nE '<E>|process = f_' log.out
grep '<E>' log.out | grep 'process = f_'
cfg PE 1 ShmDynSize -v 10       # example fix for a config error that forced restricted mode
```

## 6. Logs: where to look
| File | What it tells you |
|---|---|
| `log.out`, `sv.log`, `sv_ha_cmds.log` | Application start/stop and errors. Usually one of the three has the answer |
| `ULOG.<yymmdd>` | Tuxedo daily log |
| `/var/log/pacemaker/pacemaker.log` | Cluster manager |
| `/var/log/cluster/corosync.log` | Cluster messaging |
| `/var/log/messages` | OS |
| `catalina.out` | Tomcat (adapters, Jenkins) |

```bash
cd $ATA_DATA_SERVER_LOG
log_analyze log.out
ulog_analyze ULOG.<yymmdd>
logu; logu -sg <SERVER_GROUP>
cberr <MESSAGE_ID>              # e.g. cberr 00444 / cberr E96564
LOG f                           # append after a start command to tail the cause

# Cross-correlate an incident window
grep -HnE 'May 22 02:42:1[0-9]|Timer expired|attrd|cib|crmd|schedulerd|corosync' \
  /var/log/messages /var/log/pacemaker/pacemaker.log /var/log/cluster/corosync.log \
  $ATA_DATA_SERVER_LOG/sv_ha_cmds.log $ATA_DATA_SERVER_LOG/sv.log 2>/dev/null

grep -n 'E03622\|TPETIME\|deadlock\|TPEABORT\|queue\|lock' log.out ULOG* sv_start.out
grep -B 40 "Illegal access" catalina.out | less
```

## 7. Tuxedo and process recovery
```bash
tmadmin -v                              # Tuxedo version
tuxservers | grep -E 'RSP1100|RWDB'
ps -ef | grep '[f]_rsp'

tmshutdown -s <server>
tmboot     -s <server>
tmshutdown -s f_rsp -k TERM
tmboot -s f_rsp -g RSP1100 -i 5         # one specific process
tmboot -s trerate -i <server_id>
tmboot -g BILLING -S trerodb -i 800 -i 801
tmboot -g OCSHUB -i 1300                # restart one market's hub servers

svrestart -tux f_rsp
svrestart -tux authsvr
svrestart -bmp SFM:52; LOG f

# Background management processes
ecp "biBmpProcessStop&('TRESOAP',24)"
ecp "biBmpProcessStart&('TRESOAP',24)"
ecp "biEroStart&(<seqnr>)"              # one ERO process

# Too many idle sessions?
tuxdomain
ecp "biPing&()"
ecp 'biClientMonitor&()'
```

## 8. Adapters (REST / SOAP / Diameter)
```bash
cfg -l perest                         # list PE REST configurations
cfg PEREST 1                          # show config 1
cfg PEREST 1 INSTANCE -v SV1          # assign to an instance
ecp "biBmpProcessStop&('PEREST',1)"
ecp "biBmpProcessStart&('PEREST',1)"

# Restart REST inbound servers
for n in 1 2 3 4; do
  ecp "biTrerestInStop&($n)"; ecp "biTrerestInStart&($n)"
done

# Which instance / port does an adapter use?
cfg BAS 1 INSTANCE
cfg BAS 1 HTTP_PORT
hostname -I | cut -d' ' -f1
grep <service_name> /etc/services && netstat -ln | grep <PORT>

# Health: is each adapter process alive?
pu | grep -E 'dcea|trewfe|tresoap|perest|http2|kafka|trerest_in|trerest_out|trerate|diamsvr|mail'
cfg DIAMETER-DCEA ADAPTERS INSTANCE STATUS
```

## 9. Turning traces off
Run after troubleshooting so performance returns to normal. Use `$ATA_REL_SERVER_BIN/ecp` if `ecp` is not on the PATH.
```bash
eton  -pe_epm -sfm <duration>          # trace ON for a period
etoff -pe_epm -sfm 1                   # trace OFF

ecp "biBroadcastFunctionCall&('SFMTrace&',['OFF'],[],['sfm'],[])"
ecp "biAuthsvrTrace&('OFF')"
ecp "biTrcEpm&(0)"
ecp "biTrcEpm&(0, 'trerodb')"
ecp "biTrcTreFunc&('biService','biServiceUpdate&',1)"

# Rating processes (all, or one instance such as :1)
for p in Bkr:bkr Enm:enm Ert:ert Ero:ero Std:std Tlm:tlm; do
  name=${p%%:*}; proc=${p##*:}
  ecp "biBroadcastFunctionCall&('${name^^}Trace&',['OFF'],[],['$proc'],[])"
done
ecp "biBroadcastFunctionCall&('TRERATETrace&',['OFF'],['trerate'],[],[])"

# Billing and adapters
ecp "biTrcRental&('OFF')"
ecp "biTrcBGP&('OFF')"
ecp "biTrcIGP&('OFF')"
ecp "xmlrpc.clt.XmlRpcTrace&('com',0)"
ecp "biSMPPTrace&('com',0)"
```

## 10. Installation and release
```bash
# Single-instance installation
./install.<REL> -v -noexport -nodvp -restart \
  -option "SIZEFILE=current" -option "PostInstall_Question=y" -option "PreStartInstall_Question=y" -S 1

# Multi-instance HA (cold)
./install.<REL> -noexport -nodvp -installmode 'cold' -v -restart \
  -option 'SIZEFILE=current' -option 'PostInstall_Question=y' -option 'PreStartInstall_Question=y' -S 1

# Run detached and follow the log
nohup ./install.<REL> ... &
grep 'Register Release Information' "$(ls -1tr $ATA_HOME/admin/releases/<REL>/install.<REL>.*.log)"

# Manual deployment of a release package
cp <Release>.tar.gz "$ATA_HOME/admin/releases" && cd "$ATA_HOME/admin/releases"
tar zxvf <Release>.tar.gz && cd <Release>

# Patching
sv_stop -cluster
apply_patch -m <repo>/sv_<version>p_linux.tar.gz
ata_load -s 5 -e 16
sv_start -nocb -nope
genfast --no_cache_purge
upgrade_controlled_restart -server trerwdb
ecp 'biClusterConfigure&(0)'; sv_start -cluster
```

DB sync after a deploy:
```bash
dbsync -review
dbsync -review -novalidate -parallel -nocheckseq -e <TABLE> -e <TABLE> -online
generate_pe_views
recompile_views
db_rebuild
```

## 11. Config export / import
```bash
cfg -x config_items.ini                  # export all config items
cfg -x ALL.cfg                           # export; grep '\-d' ALL.cfg to find deletions
# vi all.ini  ->  %s/<SRC_ENV>/<DST_ENV>/gi   (retarget an export to another env)
cfg TRESOAP 24 TREPROXY_PORT -v '<value>'
cfg svmon 1 ENABLED -v 0

# Cluster metadata backup (CSV)
rt_dump -f ATA_INSTANCE.csv ATA_INSTANCE
for t in CE_PartitionResourceMapping ClusterResource ClusterStandby CustomerPartition \
         CustomerPartitionRange EventErrorPartitionMap EventErrorPartitionRange \
         EventPartitionRange InstanceGroup InstanceGroupMap InstanceStatus \
         InstanceTypeGateway ScheduleReplicationGroup; do
  da_dump -f "$t.csv" "$t"
done

# Restore
rt_load -s ATA_INSTANCE ATA_INSTANCE.csv
da_load -s -o <Table>.csv <Table>
```

## 12. Maintenance window: disable and re-enable DCEA
```bash
date
for i in $(cfg DCEA -l INSTANCE | grep -E 'SV3|SV4|SV5' | awk '{print $1}'); do
  echo "Disabling and stopping DCEA:$i"
  cfg DCEA "$i" ENABLED -v 0
  ecp "{treEventPostx&('DCEA:$i', ['DCEASetPassive&']); }"
  sleep 1
  ecp "biDceaStop&($i)"
done
date

sv_ha_ctl cluster_stop
# ... maintenance ...
sv_ha_ctl cluster_start

for i in $(cfg DCEA -l INSTANCE | grep -E 'SV3|SV4|SV5' | awk '{print $1}'); do
  echo "Enabling and starting DCEA:$i"
  cfg DCEA "$i" ENABLED -v 1
  ecp "biDceaStart&($i)"
done
```
Restart only the DCEA of the local instance:
```bash
for i in $(cfg DCEA -1 INSTANCE | awk -v inst="$ATA_INSTANCE" '$3==inst {print $1}'); do
  ecp "biDceaStop&($i)"; ecp "biDceaStart&($i)"
done
```

## 13. Pre / post install checklist
- [ ] Free space on `$ATA_HOME` and `$ATA_HOME/work` (`df -h`) meets the minimum
- [ ] No users logged in (`tuxusers -b`, `tuxusers -r`, `pe_admin -u`)
- [ ] DB and OS clocks agree (`date` vs `SELECT SYSDATE FROM DUAL`); timezone as expected
- [ ] `dbsync -review` is clean
- [ ] Config items exported (`cfg -x`)
- [ ] Filesystem backup taken and **verified** (`tar -tzf`)
- [ ] Guaranteed restore point created (see [oracle.md](oracle.md))
- [ ] After install: `sv_status -cluster`, `sv_ha_ctl cluster_status`, all adapters up, no invalid DB objects
- [ ] Restore point dropped once the release is accepted

Timezone check:
```bash
echo $TZ; date +'%d-%m-%Y %H:%M:%S'
sqlplus -S "$DB_CONNECT" <<SQL
SELECT SYSDATE FROM DUAL;
SELECT DBTIMEZONE FROM DUAL;
SQL
```

## 14. Reporting server (MySQL + Apache)
```bash
perxdb start | stop
perxweb start | stop
loader -ls                                   # list sources
loader -s "<HOST> <INSTANCE>" -ll
nohup autogen --portal -s "<HOST> <INSTANCE>" --host localhost:<PORT> -v -l <ids> &
nohup download -f <file>.manifest -s "<HOST> <INSTANCE>" --load -noprompt -v &
ps -ef | grep mysqld
```
Check `$PERX_VAR/apache/error_log` and `$PERX_CONFIG/httpd.conf` first when something breaks. Enable MySQL binary logging only for short periods; it is very verbose.
