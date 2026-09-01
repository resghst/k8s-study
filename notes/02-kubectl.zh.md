[English](02-kubectl.md) · **中文**

# kubectl

> 你會花最多時間用的命令列工具。值得好好學,而不是背幾行。

**前一課:** [架構總覽](01-architecture.zh.md)

---

# Part 1 — 操作走一遍

## 確認你指向哪裡

```bash
kubectl config current-context
```

```
default
```

只要輸出讓你意外,就**先**做這件事。對錯叢集下手,是常見又代價高昂的錯誤。

## 指令的形狀

每個指令都遵循同樣的文法:

```
kubectl <verb> <resource-type> [name] [flags]
```

```bash
kubectl get pod web -o yaml
#       verb resource name flag
```

一部分一部分試:

```bash
kubectl get pods                       # 動詞 + 資源
kubectl get pods web                   # + 指定名字
kubectl get pods web -o wide           # + 一個 flag
```

```
NAME   READY   STATUS    RESTARTS   AGE   IP           NODE     NOMINATED NODE   READINESS GATES
web    1/1     Running   0          12s   10.42.0.11   hankpc   <none>           <none>
```

## 短名稱省打字

```bash
kubectl get po        # 等同於:kubectl get pods
kubectl get ns
```

```
NAME              STATUS   AGE
default           Active   65m
kube-system       Active   65m
kube-public       Active   65m
kube-node-lease   Active   65m
```

## 問叢集,不要問搜尋引擎

有哪些資源類型?

```bash
kubectl api-resources | head -5
```

```
NAME                SHORTNAMES   APIVERSION   NAMESPACED   KIND
bindings                         v1           true         Binding
componentstatuses   cs           v1           false        ComponentStatus
configmaps          cm           v1           true         ConfigMap
endpoints           ep           v1           true         Endpoints
```

一個 Pod 有哪些欄位?

```bash
kubectl explain pod.spec.containers | head -12
```

```
KIND:       Pod
VERSION:    v1

FIELD: containers <[]Container>

DESCRIPTION:
    List of containers belonging to the pod. Containers cannot currently be
    added or removed. There must be at least one container in a Pod.
```

一個指令接受哪些 flag?

```bash
kubectl logs --help
```

每個 `--help` 結尾都有一段 **Examples**。上網查之前先讀那個。

## 只取出一個值

```bash
kubectl get pod web -o jsonpath='{.status.podIP}'
```

```
10.42.0.11
```

## 自己組一張表

```bash
kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,IMAGE:.spec.containers[0].image
```

```
NAME   NODE     IMAGE
web    hankpc   nginx:alpine
```

## 讓 kubectl 幫你產 manifest,而不是手寫

```bash
kubectl run demo --image=nginx:alpine --dry-run=client -o yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: demo
  name: demo
spec:
  containers:
  - image: nginx:alpine
    name: demo
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

什麼都沒被建立 —— `--dry-run=client` 在本機組出物件並印出來。

## 看 kubectl 真正送了什麼

```bash
kubectl get pods -v=6
```

```
I0901 02:33:23.783256  loader.go:407] Config loaded from file:  /etc/rancher/k3s/k3s.yaml
I0901 02:33:23.853796  round_trippers.go:632] "Response" verb="GET" url="https://127.0.0.1:6443/api/v1/namespaces/default/pods?limit=500" status="200 OK" milliseconds=67
NAME   READY   STATUS    RESTARTS   AGE
web    1/1     Running   0          2m6s
```

---

# Part 2 — 原理說明

## kubectl 只是一個 REST client

每個指令都變成一個 HTTP 請求:

```
kubectl get pods
  -> GET https://127.0.0.1:6443/api/v1/namespaces/default/pods
```

kubectl 沒有任何特異功能。它讀 **kubeconfig** 來知道要跟哪個叢集講話、以誰的身分認證,
送出 REST 呼叫,再把回來的 JSON 排版。kubectl 做的任何事你都能用 `curl` 做 —— kubectl
只是讓它變得能忍受。

這就是為什麼 `-v=6` 和 `-v=8` 是這麼好的除錯工具:它們讓你看到任何指令背後確切的請求
與回應。

## kubeconfig 與 context

kubectl 讀 `~/.kube/config`,或 `KUBECONFIG` 指向的檔案。一個 **context** 綁三樣東西:

- **cluster** —— API server URL 和它的 CA 憑證
- **user** —— 你的認證(憑證、token、或 exec 外掛)
- **namespace** —— 你省略 `-n` 時的預設值

切換 context 就一次切換這三個。在 k3s 上這個 context 叫 `default`,存在
`/etc/rancher/k3s/k3s.yaml`。

## 為什麼有短名稱

資源類型接受單數、複數、短寫 —— `pod`、`pods`、`po` 是同一個東西:

| 短 | 全名 | 短 | 全名 |
| --- | --- | --- | --- |
| `po` | pods | `cm` | configmaps |
| `deploy` | deployments | `pvc` | persistentvolumeclaims |
| `rs` | replicasets | `sa` | serviceaccounts |
| `svc` | services | `ing` | ingresses |
| `ns` | namespaces | `no` | nodes |

`kubectl api-resources` 是**你這個**叢集權威的清單,包含自訂資源加上去的短名稱。

## apply vs. create vs. replace

| 指令 | 行為 |
| --- | --- |
| `apply` | 不存在就建立、存在就 patch —— **冪等,用這個** |
| `create` | 已存在就失敗 |
| `replace` | 整個物件覆寫;不存在就失敗 |
| `delete` | 刪掉它 |

`apply` 是**宣告式**的:「讓叢集符合這個檔案」。編輯後再跑一次,只有差異會被送出。它會
在一個 annotation 裡記錄它套用過什麼,所以它還能偵測到你*移除*的欄位 —— 這是 `create`
和 `replace` 做不到的。

## 兩種 dry-run 模式

| 模式 | 它做什麼 |
| --- | --- |
| `--dry-run=client` | 在本機組出物件、印出來,完全不連叢集 |
| `--dry-run=server` | 送去做完整驗證與補預設值,但不寫入 |

用 `client` 產出 manifest 骨架;用 `server` 在真正 apply 前抓出結構錯誤。

## describe 不是 `get -o yaml`

`get -o yaml` 傾印一個物件。`describe` 則**把相關資訊接起來**:物件的欄位、它的擁有者、
以及最近關於它的 **Events**。東西壞掉時,Events 那段會指出原因。這就是為什麼除錯路徑
永遠從 `describe` 開始。

## 命名空間(Namespaces)

大多數資源活在某個命名空間裡;沒有 `-n` 時,kubectl 用 context 的預設值。叢集層級的
物件(nodes、namespaces、PersistentVolumes、ClusterRoles)完全忽略 `-n` ——
`kubectl api-resources --namespaced=false` 會列出它們。

---

# Part 3 — 指令參考

### Context 與叢集

```bash
kubectl config current-context
kubectl config get-contexts              # * 標記目前使用中的
kubectl config view --minify             # 目前設定,secret 遮蔽
kubectl config use-context <name>
kubectl config set-context --current --namespace=dev
kubectl cluster-info
kubectl version
```

### 探索與說明

```bash
kubectl help
kubectl <command> --help
kubectl api-resources
kubectl api-resources --namespaced=false
kubectl api-versions
kubectl explain pod
kubectl explain pod.spec.containers
kubectl explain pod --recursive
```

### 列出

```bash
kubectl get pods
kubectl get pods -o wide
kubectl get pods -A                      # 所有命名空間
kubectl get pods -n kube-system
kubectl get pods -w                      # 監看直到 Ctrl+C
kubectl get all                          # 一次列出常見的工作負載類型
```

### 輸出格式

```bash
kubectl get pods -o yaml
kubectl get pods -o json
kubectl get pods -o name                            # "pod/web"
kubectl get pod web -o jsonpath='{.status.podIP}'
kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName
```

### 篩選

```bash
kubectl get pods -l app=hello                       # 標籤 selector
kubectl get pods -l 'env in (dev,test)'             # 集合式
kubectl get pods --show-labels
kubectl get pods --field-selector status.phase=Running
kubectl get pods --sort-by=.metadata.creationTimestamp
```

### 檢視

```bash
kubectl describe pod web
kubectl describe node <name>
kubectl get events --sort-by=.lastTimestamp
kubectl get events -A --sort-by=.lastTimestamp | tail -30
kubectl get events --field-selector type=Warning
```

### 建立與變更

```bash
kubectl apply -f pod.yaml
kubectl apply -f ./manifests/
kubectl apply -k ./overlays/dev                     # kustomize
kubectl apply -f pod.yaml --dry-run=server          # 只驗證
kubectl diff -f pod.yaml                            # 預覽變更

kubectl run web --image=nginx:alpine --dry-run=client -o yaml
kubectl create deployment web --image=nginx:alpine --dry-run=client -o yaml

kubectl edit pod web
kubectl scale deploy web --replicas=3
kubectl set image deploy/web web=nginx:1.27
kubectl label pod web env=test
kubectl annotate pod web owner=hank
kubectl patch pod web -p '{"metadata":{"labels":{"tier":"front"}}}'
```

### 刪除

```bash
kubectl delete pod web
kubectl delete -f pod.yaml
kubectl delete -f pod.yaml --ignore-not-found
kubectl delete pods -l app=hello
kubectl delete pod web --grace-period=0 --force
```

### 除錯

```bash
kubectl logs web
kubectl logs web -f
kubectl logs web --previous                         # 上次重啟之前
kubectl logs web -c <container>
kubectl logs web --since=15m --tail=100
kubectl logs -l app=hello --all-containers --prefix

kubectl exec web -- env
kubectl exec -it web -- sh
kubectl debug -it web --image=busybox --target=web  # 用一個沒有 shell 的 image

kubectl port-forward pod/web 8080:80
kubectl port-forward svc/web 8080:80
kubectl cp web:/etc/nginx/nginx.conf ./nginx.conf

kubectl top nodes
kubectl top pods
```

### 幾乎到處都能用的 flag

| Flag | 意義 |
| --- | --- |
| `-n <ns>` / `--namespace` | 目標命名空間 |
| `-A` / `--all-namespaces` | 每個命名空間 |
| `-o <format>` | 輸出格式 |
| `-l <selector>` | 依標籤篩選 |
| `-f <file\|dir\|url>` | 從 manifest 讀入物件 |
| `-w` | 監看變化 |
| `--dry-run=client\|server` | 不寫入地建立/驗證 |
| `-v=6` … `-v=8` | 顯示底層的 API 呼叫 |
| `--context` / `--kubeconfig` | 為單一指令覆寫目標叢集 |

### 讓它用起來更順手

```bash
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
```

---

## 重點回顧

- kubectl 是個薄薄的 REST client;`-v=8` 顯示它確切送了什麼。
- 把 `verb resource name flags` 學會一次,每個指令都照這個走。
- `--help`、`api-resources`、`explain` **從你自己的叢集**回答大部分問題。
- 正式的東西用 `apply`,產 manifest 用 `--dry-run=client -o yaml`,`diff` 拿來預覽。
- `describe`(Events)→ `logs --previous` → `exec` 是標準除錯路徑。
- 相信眼前的輸出之前,永遠先確認你的 **context**。

---

**下一課:** [Pod](03-pods.zh.md) —— 你真正會建立的第一個物件。

[目錄](../README.md) · [疑難排解](troubleshooting.zh.md)
