# Jenkins

> Pipeline pattern: validate nodes, check disk space, then back up in parallel on every node of an environment.
> Full pipeline: [`pipelines/filesystem-backup.Jenkinsfile`](../pipelines/filesystem-backup.Jenkinsfile).

## Pipeline design
| Stage | Purpose | Failure behavior |
|---|---|---|
| 1. Validate node availability | Allocate a node with the chosen label within 20 s | Abort with a clear message |
| 2. Validate disk space on all nodes | `df -BG "$APP_HOME"` must show at least `REQUIRED_GB` free | Abort before touching anything |
| 3. Backup in parallel | One `tar.gz` per node, 3 h timeout, `trap` stops children on abort | Fails the build |
| post | Archive the per-node artifact (`ls -lh` of the backup) | Message on success / abort / failure |

Techniques worth reusing:
- `agent none` at the top and `node(label)` inside stages, so nothing holds an executor while waiting.
- Build the list of nodes from `Jenkins.instance.nodes` filtered by label, then build a `branches` map and call `parallel branches`.
- Escape shell variables inside Groovy triple-quoted strings (`\$VAR`) and let Groovy expand only the parameters (`${params.X}`).
- `returnStatus: true` turns a shell exit code into data so the pipeline can decide what to do.

## Operations
```bash
# Is Jenkins running? Restart it if not (cron-friendly)
CURR=$(ps -ef | grep jenkins | grep -v grep | wc -l)
if [[ $CURR -eq 0 ]]; then . ~/.profile > /dev/null 2>&1; ~/Startup_Jenkins.ksh; fi

# Back up jobs without build history, and plugins
tar --exclude='*/builds' -czvf jobs_nb.tar.gz jobs
tar -czvf plugins_$(date +%m%d%y).tar.gz plugins

# Push plugins to another controller
scp *.hpi <USER>@<HOST>:<JENKINS_HOME>/plugins/
```

## Guard rail: block manual release cuts
A shell function that refuses interactive use, so releases always go through Jenkins:
```bash
releaser() {
  if [[ "$RELEASER_OVERRIDE" == "true" && -t 0 ]]; then
    /path/to/releaser "$@"; return $?
  fi
  if [[ -t 0 ]]; then
    echo "Release cut needs to be done from the Jenkins console"
    return 1
  fi
  /path/to/releaser "$@"
}
```
Override for a justified manual run: `export RELEASER_OVERRIDE=true`.

## Credentials
- Store passwords, tokens and keys in **Jenkins Credentials** and reference them with `withCredentials`.
- Never hard-code a default admin account in a script or note.

## Release-install job checklist
- Pre-checks: disk space, users logged in, config export, restore point.
- Install, then post-checks (see [ha-cluster-and-app-ops.md](ha-cluster-and-app-ops.md#13-pre--post-install-checklist)).
- Notify by email on success or failure using a distribution list, not a personal address.
