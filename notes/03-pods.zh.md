[English](03-pods.md) · **中文**

# Pod

> Kubernetes 排程的最小單位,也是其他一切的基礎。

**前一課:** [kubectl](02-kubectl.zh.md)

---

# Part 1 — 操作走一遍

依序做。每個指令後面附上預期輸出。

## 跑你的第一個 Pod

```bash
kubectl run web --image=nginx:alpine
```

```
pod/web created
```

```bash
kubectl get pods
```

```
NAME   READY   STATUS              RESTARTS   AGE
web    0/1     ContainerCreating   0          0s
```

`ContainerCreating` 表示 image 還在拉。等幾秒再看一次,這次加 `-o wide`:

```bash
kubectl get pods -o wide
```

```
NAME   READY   STATUS    RESTARTS   AGE   IP           NODE     NOMINATED NODE   READINESS GATES
web    1/1     Running   0          12s   10.42.0.11   hankpc   <none>           <none>
```

`READY 1/1` 和 `Running` 表示它起來了。`-o wide` 多顯示 **Pod IP**(`10.42.0.11`)
和它落腳的**節點**(`hankpc`)。

## 看它經歷了什麼

```bash
kubectl describe pod web
```

重點在最下面:

```
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  12s   default-scheduler  Successfully assigned default/web to hankpc
  Normal  Pulled     12s   kubelet            Container image "nginx:alpine" already present on machine
  Normal  Created    12s   kubelet            Container created
  Normal  Started    12s   kubelet            Container started
```

這就是架構那一課真實上演:**scheduler** 挑了節點,接著 **kubelet** 拉 image、建立容器、
啟動它。

第一次跑的話,你還會看到一個 `Pulling` 事件和一行 `Pulled`,回報下載時間與 image 大小。

## 讀它的日誌

```bash
kubectl logs web
```

```
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
...
```

## 在它裡面跑指令

```bash
kubectl exec web -- nginx -v
```

```
nginx version: nginx/1.31.4
```

或拿一個 shell:

```bash
kubectl exec -it web -- sh
```

```
/ # curl -s localhost | head -4
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
/ # exit
```

## 取出單一欄位

```bash
kubectl get pod web -o jsonpath='{.status.podIP}'
```

```
10.42.0.11
```

## 自己寫一份 manifest

建立 [`playground/pod.yaml`](../playground/pod.yaml):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello
  labels:
    app: hello
spec:
  containers:
    - name: hello
      image: nginx:alpine
      ports:
        - containerPort: 80
```

```bash
kubectl apply -f playground/pod.yaml
```

```
pod/hello created
```

```bash
kubectl get pods --show-labels
```

```
NAME    READY   STATUS    RESTARTS   AGE   LABELS
hello   1/1     Running   0          8s    app=hello
web     1/1     Running   0          3m    run=web
```

注意 `LABELS` 的差別:你的 manifest 設了 `app=hello`,而 `kubectl run` 自動產生了
`run=web`。

> **小技巧** —— 你很少需要從零手寫 YAML:
> ```bash
> kubectl run hello --image=nginx:alpine --port=80 --dry-run=client -o yaml
> ```
> 會組出物件並印出來,*不會*送到叢集。

## 見證自癒,並找出它的界線

殺掉容器的主行程:

```bash
kubectl exec web -- kill 1
kubectl get pod web
```

```
NAME   READY   STATUS    RESTARTS     AGE
web    1/1     Running   1 (5s ago)   4m
```

`RESTARTS` 變成 1 —— kubelet **原地**重啟了容器。Pod 還是同一個 Pod、同一個 IP。

一直重複做,nginx 會一直立刻失敗,於是 kubelet 開始退避(back off):

```
NAME   READY   STATUS             RESTARTS      AGE
web    0/1     CrashLoopBackOff   4 (30s ago)   6m
```

現在改成刪掉一個 Pod:

```bash
kubectl delete pod hello
kubectl get pods
```

```
pod "hello" deleted
NAME   READY   STATUS    RESTARTS   AGE
web    1/1     Running   1          6m
```

`hello` 沒了,而且**沒有東西把它救回來**。這就是關鍵一課:kubelet 會重啟*容器*,但沒有
人會重建一個裸的 *Pod*。第 4 課的 Deployment 就是來解這個的。

## 清理

```bash
kubectl delete pod web --ignore-not-found
kubectl delete -f playground/pod.yaml --ignore-not-found
kubectl get pods
```

```
No resources found in default namespace.
```

---

# Part 2 — 原理說明

## 為什麼是 Pod,而不只是容器

大多數時候你只有一個行程要跑,而容器正好就是那個。但有時候兩個行程真的是一個單位:

- 一個 web server 加上一個幫手,把新內容拉進共用目錄
- 一個應用程式加上一個讀它檔案的 log 或 metrics 傳送器
- 一個服務加上一個在本地終結 TLS 的代理

這些配對需要**共用檔案系統**、**能用 `localhost` 互連**、**永遠落在同一台機器**、
**一起啟動與停止**。單一容器無法表達「我們是一組」。

**Pod 就是那個「一組」** —— 排程與生命週期的原子:

- 一個**網路身分**:單一 IP 和 port 空間,所以裡面的容器用 `localhost` 互相講話
- 共用的 **volume**
- **共同命運**:一起被排到同一個節點;Pod 走了,它們全都走

Kubernetes 從不單獨排程一個容器 —— 它總是把容器包進一個 Pod,即使只有一個。這樣每個
更高層的物件(ReplicaSet、Deployment、Service)都有一個一致的單位可以操作。

**大多數 Pod 只裝一個容器。** 多容器 Pod 是給上面那些緊密耦合的幫手用的 —— 也就是
sidecar 模式 —— 不是給不相干的服務用的。

## 解剖

```
Pod  (name, labels, one IP)
├── container A  ── image, command, ports, env, resources, probes
├── container B  ── (optional sidecar)
└── volumes      ── mounted into one or more containers
```

## 這個物件的欄位

```bash
kubectl get pod web -o yaml
```

| 欄位 | 意義 |
| --- | --- |
| `apiVersion: v1`、`kind: Pod` | 這是哪種物件類型 |
| `metadata.name`、`metadata.labels` | 名字;標籤 —— Service 和控制器就靠它選中這個 Pod |
| `spec.containers[].image` | 要跑哪個 image |
| `spec.containers[].ports[].containerPort` | 資訊性:行程監聽的 port |
| `spec.restartPolicy` | 容器結束時要做什麼 |
| `status.phase`、`status.podIP` | 目前階段、分配到的 IP |
| `status.conditions` | readiness 細節 |

`spec` 是你要求的;`status` 是叢集回報的 —— 跟每個其他物件一樣的分法([架構 →
spec vs. status](01-architecture.zh.md))。

## restartPolicy

控制 Pod 裡某個容器結束時,kubelet 要做什麼。

| 值 | 行為 | 典型用途 |
| --- | --- | --- |
| `Always` *(預設)* | 只要結束就重啟,不管成功或失敗 | 長時間執行的服務 |
| `OnFailure` | 只在非零結束碼時重啟 | 批次 Job |
| `Never` | 永不重啟 | 事後檢查的一次性任務 |

反覆快速失敗會產生 `CrashLoopBackOff`:kubelet 在每次嘗試前等更久(10s、20s、40s…
上限 5 分鐘),讓壞掉的容器不能把節點拖垮。

## Pod 階段(phases)

| 階段 | 意義 |
| --- | --- |
| `Pending` | 已接受,但還沒有所有容器都在跑(排程中、拉 image) |
| `Running` | 已綁到節點,至少一個容器在跑 |
| `Succeeded` | 所有容器都以 0 結束,且不會重啟 |
| `Failed` | 所有容器都終止,至少一個是非零結束 |
| `Unknown` | 節點停止回報 |

注意 `CrashLoopBackOff` 和 `ImagePullBackOff` **不是**階段 —— 它們是顯示在 `STATUS`
欄的容器狀態。參見 [疑難排解](troubleshooting.zh.md)。

## 自癒的界線在哪

| 發生了什麼 | 誰來修 | 結果 |
| --- | --- | --- |
| 容器結束 | kubelet,依 `restartPolicy` | 原地重啟;同 Pod、同 IP,`RESTARTS` 加一 |
| Pod 被刪 | 沒有人 | 永久消失 |
| 它的節點掛掉 | 沒有人 | 永久消失 |

裸 Pod 沒有擁有者,所以沒有東西會重建它。給它一個擁有者 —— 一個由 Deployment 建立與
管理的 ReplicaSet —— 就是把「一個會重啟的容器」變成「一個能存活的服務」的關鍵。

---

# Part 3 — 指令參考

### 列出與選取

```bash
kubectl get pods                                     # 基本列表
kubectl get pods -o wide                             # + IP、節點
kubectl get pods -w                                  # 即時監看變化
kubectl get pods -A                                  # 所有命名空間
kubectl get pods -l app=hello                        # 依標籤
kubectl get pods --show-labels
kubectl get pods --field-selector status.phase=Running
kubectl get pod web -o jsonpath='{.status.podIP}'    # 單一欄位
kubectl get pod web -o yaml                          # 整個物件
```

### 檢視

```bash
kubectl describe pod web                  # 欄位 + 擁有者 + Events
kubectl get events --sort-by=.lastTimestamp
```

### 日誌

```bash
kubectl logs web
kubectl logs web -f                       # 追蹤
kubectl logs web --previous               # 上次重啟之前的那個實例
kubectl logs web -c <container>           # 特定容器
kubectl logs web --since=15m --tail=100
kubectl logs -l app=hello --all-containers --prefix
```

### 進出容器

```bash
kubectl exec web -- env                   # 單一指令
kubectl exec -it web -- sh                # 互動式 shell
kubectl exec -it web -c <container> -- sh
kubectl port-forward pod/web 8080:80      # localhost:8080 -> container:80
kubectl cp web:/etc/nginx/nginx.conf ./nginx.conf
kubectl debug -it web --image=busybox --target=web   # 用一個沒有 shell 的 image
```

### 變更與移除

```bash
kubectl label pod web env=test
kubectl annotate pod web owner=hank
kubectl top pod web                       # CPU/記憶體
kubectl delete pod web
kubectl delete pod web --grace-period=0 --force
kubectl delete -f playground/pod.yaml --ignore-not-found
```

### 探索欄位

```bash
kubectl explain pod.spec.containers
kubectl explain pod.spec.containers.livenessProbe
```

---

## 重點回顧

- Pod 是排程與生命週期的**原子單位**:共用 IP、共用 volume、共同命運。單一容器是常態。
- 丟過即棄的 Pod 用 `kubectl run`,產 manifest 用 `--dry-run=client -o yaml`,正式的
  東西用 `kubectl apply -f`。
- `describe`(Events)→ `logs --previous` → `exec` 是標準除錯路徑。
- kubelet 會重啟**容器**;沒有東西會重建一個裸的 **Pod**。

---

**下一課:** [Deployment](04-deployments.zh.md) —— 給你的 Pod 一個會重建它們的擁有者。

[目錄](../README.md) · [物件參考](objects.zh.md) ·
[疑難排解](troubleshooting.zh.md)
