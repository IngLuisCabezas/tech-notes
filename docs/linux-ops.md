# Linux Operations

> Day-to-day commands for diagnosing hosts, moving files and managing SSH access.
> Placeholders: `<HOST>`, `<USER>`, `<PORT>`, `$APP_HOME`.

## Contents
- [Hardware and memory](#hardware-and-memory)
- [Ports and processes](#ports-and-processes)
- [Disk and large files](#disk-and-large-files)
- [Archives (tar)](#archives-tar)
- [SSH keys](#ssh-keys)
- [File transfer](#file-transfer)
- [Fan-out over many hosts](#fan-out-over-many-hosts)
- [HugePages check](#hugepages-check)
- [Permissions audit](#permissions-audit)
- [Mail queue (Postfix)](#mail-queue-postfix)
- [Text tricks](#text-tricks)

## Hardware and memory
```bash
lscpu -e
lscpu | grep -E '^CPU\(s\)|^Thread|^Core|^Socket'
free -h            # human readable
free -tmh          # with totals, in MB
```

## Ports and processes
```bash
# Who is listening on a port?
netstat -tuln  | grep 8080
netstat -tulnp | grep java          # needs root to show every PID
lsof -i :<PORT>
ss -ltnp | grep <PORT>              # modern replacement for netstat

# Port registry on the box
cat /etc/services | grep <service_name>

# Truncate a log without deleting the file handle
truncate -s 0 <file>.log
```

## Disk and large files
```bash
find . -type f -exec du -Sh {} + | sort -rh | head -n 10
df -h
df -h "$APP_HOME"
lsof | egrep "deleted|COMMAND"      # space held by deleted-but-open files
```

## Archives (tar)
```bash
# Create / extract
tar -cvzf backup.tar.gz <dir>
tar -xvzf backup.tar.gz
tar -zxvf archive.tar.gz

# Exclude a pattern (e.g. Jenkins build history)
tar --exclude='*/builds' -czvf jobs_nb.tar.gz jobs

# Date-stamped archive
tar -czvf plugins_$(date +%m%d%y).tar.gz plugins

# Inspect WITHOUT extracting
tar -tzf backup.tar.gz | cut -d/ -f1 | sort -u            # top-level entries
tar -tzf backup.tar.gz | grep '^admin/releases'           # exact path
tar -tf  backup.tar.gz --wildcards '*.profile*'           # one specific file
```

## SSH keys
```bash
ssh-keygen -t rsa                       # or: ssh-keygen -t ed25519
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub                   # copy this to the remote host

# On the remote host
touch ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys
vi ~/.ssh/authorized_keys               # paste the PUBLIC key
```
> Never commit `id_rsa`, `*.pem` or `*.ppk`. Public keys (`*.pub`) are safe but reveal usernames and hostnames, so keep them out of public notes as well.

## File transfer
```bash
scp <USER>@<HOST>:/remote/file /local/dest
scp -i <KEY>.pem file <USER>@<HOST>:/remote/dir

# rsync: only newer files, compressed, with progress
rsync -e /usr/bin/ssh --update -raz --progress "$APP_HOME/dir" <USER>@<HOST>:"$APP_HOME"
```

## Fan-out over many hosts
```bash
# Loop over a static list
for host in <HOST1> <HOST2> <HOST3>; do
  echo "Syncing to $host"
  rsync -e /usr/bin/ssh --update -raz --progress "$APP_HOME/work/jenkins" "<USER>@${host}:$APP_HOME/work"
done

# Run a command on every node in parallel, then wait
hosts="<HOST1> <HOST2> <HOST3>"
for h in $hosts; do
  ssh -q -n "$h" 'cd "$APP_HOME" && tar -xaf FS.tar.gz' &
done
wait

# Check that a file exists on every node
for h in $hosts; do
  ssh -q -n "$h" "[ -f /path/to/file.tar.gz ] && echo $h EXISTS || echo $h MISSING" &
done
wait

# Read hosts into a bash array
readarray -t envs <<< "$list_of_envs"
```

## HugePages check
```bash
grep HugePages_ /proc/meminfo
TOTAL=$(grep HugePages_Total /proc/meminfo | awk '{print $2}')
FREE=$(grep HugePages_Free  /proc/meminfo | awk '{print $2}')
[ "$TOTAL" -eq "$FREE" ] && echo "HugePages NOT in use" || echo "HugePages in use"
```

## Permissions audit
```bash
# Find world-writable files (count first!)
find -L "$APP_HOME" -perm 0777 | wc -l
find -L "$APP_HOME" -perm 0777 -ls

# Fix only what you own: 777 -> 755
find -L "$APP_HOME/work" \( -type d -o -type f \) -perm 0777 \
     -user "$(whoami)" -group "$PROJECT_GROUP" -exec chmod 755 {} +
```

## Mail queue (Postfix)
```bash
systemctl status postfix
postqueue -p                  # show queue
postsuper -d ALL              # purge the whole queue
sudo systemctl restart postfix
cat /var/log/maillog

# Send a test mail
mailx -s "Test" -r "Sender <sender@example.com>" recipient@example.com <<EOF
Hi World.
EOF
```

## Text tricks
```bash
sed -i 's/OLD_NAME/NEW_NAME/g' file.txt         # in-place replace
# vi: comment a range            :3020,3030s/^/# /
echo -e "a\tb\nc"                               # -e enables \n \t escapes
```
