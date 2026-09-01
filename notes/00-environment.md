**English** · [中文](00-environment.zh.md)

# Setting Up a Practice Cluster

> Build a throwaway Kubernetes cluster you can break and rebuild freely:
> **k3s inside WSL2 Ubuntu**.

**You need:** Windows with WSL2 and a Linux distro installed.

---

# Part 1 — Walkthrough

Everything here runs **inside WSL**, not PowerShell. Open it with:

```bash
wsl
```

Your prompt changes from `PS C:\...>` to `hank@hankPC:~$`.

## Check the prerequisites

k3s installs as a systemd service, so the distro needs systemd enabled:

```bash
cat /etc/wsl.conf
```

```
[boot]
systemd=true

[network]
generateResolvConf = true
```

```bash
ps -p 1 -o comm=
```

```
systemd
```

```bash
systemctl is-system-running
```

```
running
```

`degraded` is also fine. If systemd is **not** running, add the `[boot]` section
above to `/etc/wsl.conf`, run `wsl --shutdown` from PowerShell, and reopen WSL.

## Install k3s

```bash
curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" sh -
```

```
[INFO]  Finding release for channel stable
[INFO]  Using v1.36.4+k3s1 as release
[INFO]  Downloading hash ...
[INFO]  Downloading binary ...
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
[INFO]  systemd: Starting k3s
```

It will ask for your sudo password.

## Point kubectl at the cluster

```bash
echo 'export KUBECONFIG=/etc/rancher/k3s/k3s.yaml' >> ~/.bashrc
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

The first line makes it permanent; the second applies it to the current shell.

## Verify

```bash
kubectl get nodes
```

```
NAME     STATUS   ROLES           AGE   VERSION
hankpc   Ready    control-plane   64m   v1.36.4+k3s1
```

One node, `Ready`, with the `control-plane` role — on a single-machine cluster
that node is both the control plane and the worker.

```bash
kubectl get nodes -o wide
```

```
NAME     STATUS   ROLES           AGE  VERSION        INTERNAL-IP     OS-IMAGE             KERNEL-VERSION                      CONTAINER-RUNTIME
hankpc   Ready    control-plane   64m  v1.36.4+k3s1   172.29.223.79   Ubuntu 22.04.3 LTS   6.18.33.2-microsoft-standard-WSL2    containerd://2.3.4-k3s1.36
```

Note `containerd` as the runtime — that is the piece doing the job your Docker
daemon used to do.

```bash
kubectl cluster-info
```

```
Kubernetes control plane is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy
```

```bash
kubectl get pods -A
```

```
NAMESPACE     NAME                                      READY   STATUS      RESTARTS      AGE
kube-system   coredns-54996dc9b4-qqtc5                  1/1     Running     0             64m
kube-system   helm-install-traefik-crd-4rxfc            0/1     Completed   0             64m
kube-system   helm-install-traefik-q5cgv                0/1     Completed   2 (63m ago)   64m
kube-system   local-path-provisioner-77b9867795-t4h6m   1/1     Running     0             64m
kube-system   metrics-server-6dc596dfb8-d8t7w           1/1     Running     0             64m
kube-system   svclb-traefik-623d8f1b-wqbfs              2/2     Running     0             62m
kube-system   traefik-59b7647586-w4ckg                  1/1     Running     0             62m
```

These are the add-ons k3s bundles, all in the `kube-system` namespace. The
`helm-install-*` entries showing `Completed` are **one-shot Jobs that already
finished** — not errors.

**Your cluster is ready.**

---

# Part 2 — How it works

## Why this setup

Learning Kubernetes requires a cluster you are not afraid to destroy. Four ways to
get one:

| Option | Verdict |
| --- | --- |
| **Managed cloud** (GKE / EKS / AKS) | Closest to production, but costs money, needs a cloud account, and is slow to reset. Good later. |
| **Docker Desktop** | Works, but needs a paid licence at larger companies. |
| **Rancher Desktop** | Tried first here; its WSL integration failed on this machine (`/sbin/init exited with status 1` — endpoint-protection software blocking the WSL image import). |
| **k3s in an existing WSL2 distro** | ✅ **Chosen.** One binary, no GUI, no extra WSL distro, nothing for security software to interfere with. |

## What k3s is

k3s is a **fully conformant** Kubernetes distribution packaged for a single
machine. One ~70 MB binary contains:

- the API server, scheduler, and controller-manager
- the kubelet and kube-proxy
- containerd (container runtime)
- Flannel (networking), Traefik (ingress), CoreDNS, and a `local-path` storage
  provisioner

It swaps external etcd for an **embedded SQLite** datastore. Every `kubectl`
command behaves exactly as on a full multi-node cluster — only the packaging
differs. See [Architecture → on k3s](01-architecture.md#what-this-looks-like-on-k3s).

## What the installer did

| Action | Where |
| --- | --- |
| Installed the binary | `/usr/local/bin/k3s` |
| Symlinked the CLI tools | `kubectl`, `crictl`, `ctr` → `k3s` |
| Created a systemd unit | `/etc/systemd/system/k3s.service` (enabled + started) |
| Wrote a kubeconfig | `/etc/rancher/k3s/k3s.yaml` |
| Added helper scripts | `k3s-killall.sh`, `k3s-uninstall.sh` |

`K3S_KUBECONFIG_MODE="644"` made that kubeconfig readable by your normal user
instead of root only — without it, every `kubectl` needs `sudo`.

## What KUBECONFIG does

kubectl has no built-in idea of where your cluster is. It reads a **kubeconfig**
file describing the cluster address, the CA certificate, and your credentials.
`KUBECONFIG=/etc/rancher/k3s/k3s.yaml` points it at the one k3s wrote, whose
server is `https://127.0.0.1:6443`.

---

# Part 3 — Operating the cluster

| Task | Command |
| --- | --- |
| Check the service | `sudo systemctl status k3s` |
| Stop the cluster | `sudo systemctl stop k3s` |
| Start it again | `sudo systemctl start k3s` |
| Restart | `sudo systemctl restart k3s` |
| Follow logs | `sudo journalctl -u k3s -f` |
| Kill all workloads, keep k3s | `sudo /usr/local/bin/k3s-killall.sh` |
| Uninstall completely | `/usr/local/bin/k3s-uninstall.sh` |

The systemd unit is enabled, so after `wsl --shutdown` the cluster starts again by
itself when the distro next boots.

## Optional: run kubectl from Windows

WSL2 forwards `localhost`, so a Windows-side kubectl reaches the same cluster:

```powershell
winget install -e --id Kubernetes.kubectl
mkdir $env:USERPROFILE\.kube -Force
wsl cat /etc/rancher/k3s/k3s.yaml | Out-File -Encoding ascii $env:USERPROFILE\.kube\config
kubectl get nodes
```

## Known noise on this machine

Harmless — safe to ignore:

- `InvalidDiskCapacity` / `invalid capacity 0 on image filesystem` on the node at
  startup — cosmetic, known for k3s-in-WSL.
- `FailedToCreateEndpoint` for `traefik` during first boot — a startup race;
  Traefik ends up `Running`.
- Git Bash printing `$'\377\376export': command not found` at launch — an
  unrelated `C:\Users\<you>\.bashrc` saved as UTF-16. Re-save as UTF-8 or delete
  it. The k3s `KUBECONFIG` line lives in the **WSL** `~/.bashrc`, not that one.

---

## Recap

- k3s gives you a real, conformant cluster from one binary.
- It runs as a **systemd service** — `systemctl` and `journalctl` manage it.
- `KUBECONFIG=/etc/rancher/k3s/k3s.yaml` is what connects kubectl to it.
- Run `kubectl` **inside WSL** unless you copy the kubeconfig to Windows.

---

**Next:** [Architecture Overview](01-architecture.md) — what all those moving
parts actually are.

[Contents](../README.md)
