# Tech Notes

> A senior engineer's field notebook: the commands, runbooks and mental models I reach for when operating Linux hosts, HA clusters, Oracle, Jenkins, Kubernetes and cloud platforms.

![Type](https://img.shields.io/badge/type-knowledge%20base-blue)
![Format](https://img.shields.io/badge/format-markdown-informational)
![Secrets](https://img.shields.io/badge/secrets-none-brightgreen)

Everything here is written to be **copied, adapted and run**. Anything environment-specific is a `<PLACEHOLDER>`.

---

## Map

| Topic | What you will find | Level |
|---|---|---|
| [Linux operations](docs/linux-ops.md) | Diagnostics, ports, tar, SSH keys, rsync, multi-host loops | Daily |
| [HA cluster and application runbook](docs/ha-cluster-and-app-ops.md) | Pacemaker/Corosync, stop/start, Tuxedo recovery, traces, release install, checklists | Incident / change |
| [Oracle](docs/oracle.md) | Restore points, flashback, health queries, scripting, password rotation | Change |
| [Jenkins](docs/jenkins.md) · [Jenkinsfile](pipelines/filesystem-backup.Jenkinsfile) | Parallel multi-node backup pipeline, guard rails | CI/CD |
| [Git and GitHub Actions](docs/git-and-github-actions.md) | Branch, commit, merge, tag, Actions snippets, pre-publish checks | Daily |
| [Kubernetes and Docker](docs/kubernetes-and-docker.md) | Concepts, `kubectl`, manifests, rollouts, Minikube | Learning / daily |
| [Google Cloud and GKE](docs/gcp-gke.md) | Org policy, cluster credentials, CI service account | Setup |
| [SSH tunnels and Windows](docs/ssh-tunnels-and-windows.md) | Port forwarding, scp, PowerShell checks | Daily |
| [Networking reference](docs/networking-reference.md) | Essential ports, OSI model, quick diagnosis | Reference |

## Repository layout

```text
tech-notes/
├── README.md
├── docs/            # one file per topic
├── pipelines/       # reusable pipeline code
└── assets/          # diagrams and cheat sheets
```

## Conventions

| Convention | Meaning |
|---|---|
| `<PLACEHOLDER>` | Replace before running |
| `$APP_HOME`, `$ATA_HOME` | Application home directory of the environment owner |
| `SV1..SVn` | Application instances in a multi-instance cluster |
| `# comment` | Explains the line below it |
| **Destructive** | Command removes data or stops service. Read twice |

## Working principles

1. **Observe before acting**: status, logs and disk space first.
2. **Make the change reversible**: backup, restore point, config export.
3. **Escalate gently**: graceful stop, then `-kill`, then `-kill -kill`.
4. **Verify after every step**, not only at the end.
5. **Automate what you repeat three times**, and keep humans in the approval path.

## Security policy for these notes

- No passwords, tokens, private keys, service-account files, real hostnames, IP addresses, customer names or personal data belong in this repository.
- Use environment variables, a secrets manager or CI secrets. Reference them as `$VAR` or `<PLACEHOLDER>`.
- `.gitignore` blocks common secret file types (`*.pem`, `key.json`, `.env`, ...).
- Before pushing: `gitleaks detect --source . --log-opts="--all"`.

## Disclaimer

Notes reflect personal experience. Vendor-specific tools (`sv_*`, `ecp`, `cfg`, `da_dump`, Tuxedo, Pacemaker) behave according to your product version. Test in a non-production environment first.
