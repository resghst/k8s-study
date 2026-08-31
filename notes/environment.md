# Environment Setup

How the local cluster used throughout these notes was built: **k3s running
inside a WSL2 Ubuntu distro on Windows**. This page explains the choices, the
install, and the day-to-day commands.

## Why this setup

Learning Kubernetes needs a cluster you can break and rebuild freely. The
options, and why this one:

| Option | Verdict |
| --- | --- |
| **Managed cloud** (GKE / EKS / AKS) | Closest to production, but costs money, needs a cloud account, and is slower to reset. Good later, not for first steps. |
| **Docker Desktop + Kubernetes** | Works, but the desktop app needs a paid licence for larger companies. Skipped to keep the setup licence-free. |
| **Rancher Desktop** | Tried first. Its WSL integration failed on this machine — `/sbin/init exited with status 1`, endpoint-protection software interfering with the WSL image import. Not worth fighting. |
| **k3s directly in an existing WSL2 distro** | ✅ Chosen. One binary, no GUI, no extra WSL distro, nothing for security software to block. |

### What k3s is

k3s is a **fully conformant** Kubernetes distribution packaged for single-machine
and edge use. One ~70 MB binary bundles:

- the API server, scheduler, and controller-manager
- the kubelet and kube-proxy
- a container runtime (containerd)
- a CNI plugin (Flannel), an ingress controller (Traefik), CoreDNS, and a
  `local-path` storage provisioner

It replaces external **etcd** with an **embedded SQLite** datastore by default.
Everything you type as `kubectl` behaves exactly as on a multi-node cluster —
only the packaging differs. See
[Architecture overview §8](00-architecture.md#8-on-a-local-cluster-k3s).

## Prerequisites

- Windows 10/11 with **WSL2** and a Linux distro installed (Ubuntu 22.04 here).
- **systemd enabled** in that distro — k3s installs itself as a systemd service.

Check systemd:

```bash
cat /etc/wsl.conf        # want a [boot] section with systemd=true
ps -p 1 -o comm=         # want: systemd
systemctl is-system-running   # want: running (or degraded)
```

If systemd is not on, add this to `/etc/wsl.conf`, then run `wsl --shutdown`
from PowerShell and reopen the distro:

```ini
[boot]
systemd=true
```

## Install k3s

Run inside the WSL distro:

```bash
curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" sh -
```

What the installer does:

- downloads `k3s` to `/usr/local/bin/k3s`
- creates symlinks `kubectl`, `crictl`, `ctr` → `k3s`
- writes and enables the `k3s.service` systemd unit, then starts it
- writes a kubeconfig to `/etc/rancher/k3s/k3s.yaml`

`K3S_KUBECONFIG_MODE="644"` makes that kubeconfig readable by your normal user
instead of root-only.

## Point kubectl at the cluster

```bash
echo 'export KUBECONFIG=/etc/rancher/k3s/k3s.yaml' >> ~/.bashrc
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

The kubeconfig's server is `https://127.0.0.1:6443`.

## Verify

```bash
kubectl get nodes
kubectl get pods -A
kubectl cluster-info
```

Expected:

- one node, `STATUS=Ready`, `ROLES=control-plane`
- in `kube-system`: `coredns`, `local-path-provisioner`, `metrics-server`,
  `traefik`, `svclb-traefik-*` all `Running`
- `helm-install-traefik-*` showing `Completed` — those are one-shot Jobs, not a
  problem

## Day-to-day operations

| Task | Command |
| --- | --- |
| Service status | `sudo systemctl status k3s` |
| Stop the cluster | `sudo systemctl stop k3s` |
| Start the cluster | `sudo systemctl start k3s` |
| Restart | `sudo systemctl restart k3s` |
| Follow logs | `sudo journalctl -u k3s -f` |
| Kill all workloads, keep k3s installed | `sudo /usr/local/bin/k3s-killall.sh` |
| Uninstall k3s completely | `/usr/local/bin/k3s-uninstall.sh` |

After a `wsl --shutdown`, k3s starts again automatically the next time the distro
starts, because the systemd unit is enabled.

## Optional: drive it from Windows

The cluster runs in WSL, but `kubectl` on the Windows side can reach it because
WSL2 forwards `localhost`:

```powershell
winget install -e --id Kubernetes.kubectl
mkdir $env:USERPROFILE\.kube -Force
wsl cat /etc/rancher/k3s/k3s.yaml | Out-File -Encoding ascii $env:USERPROFILE\.kube\config
kubectl get nodes
```

## Gotchas seen on this machine

- `InvalidDiskCapacity` / `invalid capacity 0 on image filesystem` warning on the
  node at startup — cosmetic, known for k3s-in-WSL.
- `FailedToCreateEndpoint` for `traefik` during first boot — a startup race;
  Traefik ends up `Running`.
- A stray `C:\Users\<you>\.bashrc` saved as UTF-16 makes Git Bash print
  `$'\377\376export': command not found` on every launch. Unrelated to k3s; re-save
  that file as UTF-8 or delete it. The k3s `KUBECONFIG` line lives in the WSL
  distro's `~/.bashrc`, not this one.
