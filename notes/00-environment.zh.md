[English](00-environment.md) · **中文**

# 架設練習叢集

> 建一個可以隨便弄壞、重建的 Kubernetes 叢集:**WSL2 Ubuntu 裡的 k3s**。

**你需要:** 裝好 WSL2 和一個 Linux 發行版的 Windows。

---

# Part 1 — 操作走一遍

這裡所有指令都在 **WSL 裡面**跑,不是 PowerShell。用這個打開:

```bash
wsl
```

提示字元會從 `PS C:\...>` 變成 `hank@hankPC:~$`。

## 檢查前置條件

k3s 以 systemd 服務的形式安裝,所以發行版要開啟 systemd:

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

顯示 `degraded` 也沒關係。如果 systemd **沒有**在跑,把上面的 `[boot]` 段落加進
`/etc/wsl.conf`,在 PowerShell 執行 `wsl --shutdown`,再重開 WSL。

## 安裝 k3s

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

過程中會要你輸入 sudo 密碼。

## 讓 kubectl 指向叢集

```bash
echo 'export KUBECONFIG=/etc/rancher/k3s/k3s.yaml' >> ~/.bashrc
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

第一行讓它永久生效;第二行套用到目前這個 shell。

## 驗證

```bash
kubectl get nodes
```

```
NAME     STATUS   ROLES           AGE   VERSION
hankpc   Ready    control-plane   64m   v1.36.4+k3s1
```

一個節點、`Ready`、角色是 `control-plane` —— 在單機叢集上,這個節點同時是控制平面
也是工人(worker)。

```bash
kubectl get nodes -o wide
```

```
NAME     STATUS   ROLES           AGE  VERSION        INTERNAL-IP     OS-IMAGE             KERNEL-VERSION                      CONTAINER-RUNTIME
hankpc   Ready    control-plane   64m  v1.36.4+k3s1   172.29.223.79   Ubuntu 22.04.3 LTS   6.18.33.2-microsoft-standard-WSL2    containerd://2.3.4-k3s1.36
```

注意執行環境(runtime)是 `containerd` —— 它就是在做你以前 Docker daemon 做的那件事。

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

這些是 k3s 內建的附加元件,全都在 `kube-system` 命名空間。顯示 `Completed` 的
`helm-install-*` 是**已經跑完的一次性 Job** —— 不是錯誤。

**你的叢集準備好了。**

---

# Part 2 — 原理說明

## 為什麼選這個組合

學 Kubernetes 需要一個你不怕弄壞的叢集。取得叢集有四種方式:

| 選項 | 評語 |
| --- | --- |
| **雲端託管**(GKE / EKS / AKS) | 最接近正式環境,但要花錢、要雲端帳號、重置慢。之後再用不遲。 |
| **Docker Desktop** | 能用,但在較大的公司需要付費授權。 |
| **Rancher Desktop** | 這裡先試了它;它的 WSL 整合在這台機器上失敗(`/sbin/init exited with status 1` —— 端點防護軟體擋掉了 WSL 映像匯入)。 |
| **在既有的 WSL2 發行版裡跑 k3s** | ✅ **採用。** 一個 binary、沒有 GUI、不用另開 WSL 發行版,也沒有東西讓資安軟體去干擾。 |

## k3s 是什麼

k3s 是一個**完全符合規範(conformant)**的 Kubernetes 發行版,打包成單機可用。一個
約 70 MB 的 binary 就包含了:

- API server、scheduler、controller-manager
- kubelet 和 kube-proxy
- containerd(容器執行環境)
- Flannel(網路)、Traefik(ingress)、CoreDNS,以及一個 `local-path` 儲存供應器

它把外部 etcd 換成**內嵌的 SQLite** 資料庫。每一個 `kubectl` 指令的行為都跟完整的
多節點叢集完全一樣 —— 只有打包方式不同。參見
[架構 → 在 k3s 上長什麼樣](01-architecture.zh.md)。

## 安裝程式做了什麼

| 動作 | 位置 |
| --- | --- |
| 安裝 binary | `/usr/local/bin/k3s` |
| 把 CLI 工具做成符號連結 | `kubectl`、`crictl`、`ctr` → `k3s` |
| 建立 systemd 單元 | `/etc/systemd/system/k3s.service`(已啟用 + 已啟動) |
| 寫出 kubeconfig | `/etc/rancher/k3s/k3s.yaml` |
| 加入輔助腳本 | `k3s-killall.sh`、`k3s-uninstall.sh` |

`K3S_KUBECONFIG_MODE="644"` 讓那個 kubeconfig 你一般使用者也讀得到,而不是只有
root —— 沒有它的話,每個 `kubectl` 都要加 `sudo`。

## KUBECONFIG 的作用

kubectl 本身不知道你的叢集在哪。它會讀一個 **kubeconfig** 檔,裡面描述叢集位址、CA
憑證、以及你的認證資訊。`KUBECONFIG=/etc/rancher/k3s/k3s.yaml` 讓它指向 k3s 寫出的
那一個,其 server 是 `https://127.0.0.1:6443`。

---

# Part 3 — 操作叢集

| 工作 | 指令 |
| --- | --- |
| 檢查服務狀態 | `sudo systemctl status k3s` |
| 停止叢集 | `sudo systemctl stop k3s` |
| 重新啟動 | `sudo systemctl start k3s` |
| 重啟 | `sudo systemctl restart k3s` |
| 追蹤日誌 | `sudo journalctl -u k3s -f` |
| 清掉所有工作負載、保留 k3s | `sudo /usr/local/bin/k3s-killall.sh` |
| 完整解除安裝 | `/usr/local/bin/k3s-uninstall.sh` |

systemd 單元已啟用,所以 `wsl --shutdown` 之後,發行版下次開機時叢集會自己再起來。

## 選用:從 Windows 跑 kubectl

WSL2 會轉發 `localhost`,所以 Windows 端的 kubectl 也連得到同一個叢集:

```powershell
winget install -e --id Kubernetes.kubectl
mkdir $env:USERPROFILE\.kube -Force
wsl cat /etc/rancher/k3s/k3s.yaml | Out-File -Encoding ascii $env:USERPROFILE\.kube\config
kubectl get nodes
```

## 這台機器上已知的雜訊

無害 —— 可以安心忽略:

- 節點啟動時的 `InvalidDiskCapacity` / `invalid capacity 0 on image filesystem`
  —— 只是外觀訊息,k3s-in-WSL 已知現象。
- 第一次開機時 `traefik` 的 `FailedToCreateEndpoint` —— 啟動時的競態;Traefik 最後
  會變成 `Running`。
- Git Bash 啟動時印出 `$'\377\376export': command not found` —— 是無關的
  `C:\Users\<你>\.bashrc` 被存成 UTF-16 造成的。重存成 UTF-8 或刪掉即可。k3s 的
  `KUBECONFIG` 那行是在 **WSL** 的 `~/.bashrc`,不是那一個。

---

## 重點回顧

- k3s 用一個 binary 就給你一個真正、符合規範的叢集。
- 它以 **systemd 服務**執行 —— 用 `systemctl` 和 `journalctl` 管理它。
- `KUBECONFIG=/etc/rancher/k3s/k3s.yaml` 就是讓 kubectl 連上它的關鍵。
- 除非你把 kubeconfig 複製到 Windows,否則 `kubectl` 都在 **WSL 裡面**跑。

---

**下一課:** [架構總覽](01-architecture.zh.md) —— 那些零件到底是什麼。

[目錄](../README.md)
