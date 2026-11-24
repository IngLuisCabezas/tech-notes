# SSH Tunnels, SCP and PowerShell

> Reach services that are only exposed inside a remote host or cluster.
> Placeholders: `<KEY>.pem`, `<USER>`, `<EC2_PUBLIC_IP>`. Keys live **outside** any repository (for example `~/.ssh/`).

## Local port forwarding
`-L <LOCAL_PORT>:<DESTINATION_HOST>:<DESTINATION_PORT>`

```powershell
# Interactive login + tunnel (e.g. Minikube API behind an EC2 host)
ssh -i C:\keys\<KEY>.pem `
  -L 443:<MINIKUBE_IP>:443 `
  <USER>@<EC2_PUBLIC_IP>

# Tunnel only, no shell (-N)
ssh -i C:\keys\<KEY>.pem `
  -N -L 80:localhost:80 `
  <USER>@<EC2_PUBLIC_IP>

# Several services at once (NodePorts of an ingress controller)
ssh -i C:\keys\<KEY>.pem `
  -L 8080:localhost:<NODEPORT_HTTP> `
  -L 8443:localhost:<NODEPORT_HTTPS> `
  <USER>@<EC2_PUBLIC_IP>
```
```bash
# Same thing from bash
ssh -i <KEY>.pem \
  -L 8080:localhost:<NODEPORT_HTTP> \
  -L 8443:localhost:<NODEPORT_HTTPS> \
  <USER>@<EC2_PUBLIC_IP>
```
Then browse `http://localhost:8080`. PuTTY is not required; the OpenSSH client built into Windows is enough.

## Copying files
```powershell
scp -i C:\keys\<KEY>.pem .\project.zip <USER>@<EC2_PUBLIC_IP>:/home/<USER>/workspace/

# Several files at once (PowerShell line continuation = backtick)
scp "part-1.zip" `
    "part-2.zip" `
    "part-3.zip" `
    <USER>@<HOST>:/home/<USER>/workspace
```

## Connectivity checks (Windows)
```powershell
nslookup <HOSTNAME>
Test-NetConnection <HOSTNAME> -Port 443
route print
```

## Keep a session from timing out
Sends a harmless key press every 4 minutes. Use only where your policy allows it.
```powershell
Add-Type -AssemblyName System.Windows.Forms
while ($true) { [System.Windows.Forms.SendKeys]::SendWait('.'); Start-Sleep -Seconds 240 }
```

## Handy
- `Win + R` opens the Run dialog.
- Restrict key permissions on Linux: `chmod 600 <KEY>.pem` (SSH refuses keys that are world-readable).
