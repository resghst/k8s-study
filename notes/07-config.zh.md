[English](07-config.md) · **中文**

# ConfigMap & Secret

> 在執行期把設定與憑證餵進 Pod,而不是把它們烤進容器 image。

**前一課:** [Ingress](06-ingress.zh.md)

到目前為止設定都是寫死的 —— nginx image 內建的頁面,用 `kubectl exec` 手動改。真實的 app
需要各種設定(資料庫 URL、功能開關)和憑證(密碼、API key),而這些**不該**放在 image 裡:
改一個就得重 build、重部署,而且任何烤進去的密碼都會外洩給任何能 pull image 的人。解法是
把設定跟 image 解耦:

- **ConfigMap** —— 非機密設定(URL、開關、設定檔)。
- **Secret** —— 敏感資料(密碼、token、憑證)。

Pod 在啟動時讀它們。改設定 = 改 ConfigMap,不用重 build。

---

# Part 1 — 操作走一遍

依序做。

## 建立一個 ConfigMap 和一個 Secret

```bash
kubectl create configmap app-config \
  --from-literal=APP_COLOR=blue \
  --from-literal=APP_MODE=production
kubectl create secret generic app-secret \
  --from-literal=DB_PASSWORD=s3cr3t
```

```
configmap/app-config created
secret/app-secret created
```

## 看看它們 —— Secret 並沒有加密

```bash
kubectl get configmap app-config -o yaml | grep -A3 '^data:'
```

```
data:
  APP_COLOR: blue
  APP_MODE: production
```

```bash
kubectl get secret app-secret -o yaml | grep -A2 '^data:'
```

```
data:
  DB_PASSWORD: czNjcjN0
```

ConfigMap 存的是明碼。Secret 的值看起來被打亂了 —— 但那只是 **base64**,不是加密。任何有
讀取權限的人都能輕易解碼:

```bash
kubectl get secret app-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d; echo
```

```
s3cr3t
```

Secret 跟 ConfigMap 的差別在**意圖與處理方式** —— Kubernetes 用 RBAC 把它擋著、不寫進
log、還能在 etcd 裡靜態加密它 —— 但這個物件本身**不會**加密你的資料。絕不要把帶著真實
值的 Secret manifest commit 進 git。

## 在 Pod 裡消費它們 —— 兩種方式

[`playground/pod-config.yaml`](../playground/pod-config.yaml) 同時把值注入成**環境變數**、
並把 ConfigMap 掛成**檔案**。

```bash
kubectl apply -f playground/pod-config.yaml
```

當環境變數:

```bash
kubectl exec consumer -- printenv APP_COLOR DB_PASSWORD
```

```
blue
s3cr3t
```

`APP_COLOR` 來自 ConfigMap,`DB_PASSWORD` 來自 Secret —— 而且 Secret 在進容器的路上被
**自動解碼**了(你拿到 `s3cr3t`,不是 base64)。

當掛載的檔案:

```bash
kubectl exec consumer -- sh -c 'ls /etc/config; cat /etc/config/APP_COLOR; cat /etc/config/APP_MODE'
```

```
APP_COLOR
APP_MODE
blue
production
```

每個 ConfigMap key 都變成 `/etc/config` 下的一個**檔案**,值就是檔案內容。同一份
ConfigMap,依 app 期望的形式,可以當變數、也可以當檔案消費。

## 清理

```bash
kubectl delete pod consumer
kubectl delete configmap app-config
kubectl delete secret app-secret
```

---

# Part 2 — 原理說明

## 設定活在 Pod 外面,用名字引用

一個 ConfigMap 或 Secret 是 API 裡獨立的物件。Pod 不包含那些值 —— 它用名字**引用**那個
物件(`configMapKeyRef`、`secretKeyRef`、`configMap:` volume)。這跟 Service 用標籤選 Pod
是同一種解耦模式:設定和工作負載是分開的物件,靠引用接起來。一份 ConfigMap 能餵很多 Pod;
更新它,就更新了所有 Pod 的來源。

## 兩種注入方式,以及更新的陷阱

| 方式 | 呈現形式 | ConfigMap/Secret 改變時 |
| --- | --- | --- |
| **環境變數**(`valueFrom` / `envFrom`) | 行程的環境變數 | **凍結** —— Pod 必須重啟才看得到新值 |
| **Volume 掛載**(`configMap:` / `secret:` volume) | 目錄裡的檔案 | **自動更新**(kubelet 約一分鐘內重新同步) |

環境變數在容器啟動時讀一次、之後不再回頭看,所以改了值要用
`kubectl rollout restart deployment/<name>` 才會生效。掛載的 volume 由 kubelet 保持同步,
所以一個會重讀設定檔的 app 不用重啟就能拿到變更。依你需不需要熱重載來選方式。

## 什麼時候用哪種 key 風格

- **`--from-literal=KEY=value`**(或 `data:` key/value 配對)→ 最適合當**環境變數**消費,
  一個設定一個 key。
- **`--from-file=app.conf`**(或多行的值)→ 整個檔案變成一個 key,最適合**掛成檔案**讓
  app 讀。參見 [`playground/configmap.yaml`](../playground/configmap.yaml) 裡的 `app.conf`
  key。

## stringData vs data

在一個 Secret manifest 裡,`stringData` 接受**明碼**,Kubernetes 在 apply 時把它 base64
編碼進 `data` —— 所以你寫 `DB_PASSWORD: s3cr3t`,不是 base64。讀回 Secret 時永遠顯示編碼
過的 `data`。[`playground/secret.yaml`](../playground/secret.yaml) 用了 `stringData`
(而且正是為了上面 git 的理由被標為 demo-only)。

## 讓 Secret 真的保密

因為 base64 不是保護,真實專案**不會**把 Secret 的值留在 repo 裡。常見做法:在外部建立
它們(`kubectl create secret`)、啟用 **etcd 靜態加密**、或用一個 operator 從真正的金庫
注入(sealed-secrets、external-secrets、或雲端供應商的 secret manager)。Pod spec 一樣只是
用名字引用 Secret —— 只是它的來源改變了。

---

# Part 3 — 指令參考

### 建立

```bash
# ConfigMap
kubectl create configmap app-config --from-literal=APP_COLOR=blue
kubectl create configmap app-config --from-file=app.conf          # 整個檔案
kubectl create configmap app-config --from-env-file=app.env       # KEY=VALUE 各行

# Secret
kubectl create secret generic app-secret --from-literal=DB_PASSWORD=s3cr3t
kubectl create secret tls my-tls --cert=tls.crt --key=tls.key      # 給 Ingress 用
```

### 檢視

```bash
kubectl get configmap app-config -o yaml
kubectl get secret app-secret -o yaml
kubectl get secret app-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d   # 解碼
kubectl describe configmap app-config
```

### 消費(manifest 片段)

```yaml
# 一個 key 當環境變數
env:
  - name: APP_COLOR
    valueFrom:
      configMapKeyRef: { name: app-config, key: APP_COLOR }

# 一次把每個 key 都當環境變數
envFrom:
  - configMapRef: { name: app-config }
  - secretRef:    { name: app-secret }

# 掛成檔案
volumes:
  - name: config-vol
    configMap: { name: app-config }
volumeMounts:
  - { name: config-vol, mountPath: /etc/config, readOnly: true }
```

### 套用新值

```bash
kubectl apply -f playground/configmap.yaml     # 更新 ConfigMap
kubectl rollout restart deployment/web         # 若當環境變數消費則需要
```

### 探索欄位

```bash
kubectl explain pod.spec.containers.envFrom
kubectl explain pod.spec.volumes.configMap
```

---

## 重點回顧

- **ConfigMap**(非機密)和 **Secret**(敏感)把設定跟 image 解耦 —— 不用重 build 就能
  改設定。
- Pod 用名字**引用**它們;一份 ConfigMap/Secret 能餵很多 Pod。
- 把它們注入成**環境變數**(啟動時凍結 —— 要重啟才更新)或**掛成檔案**(自動同步 ——
  適合熱重載)。
- Secret 是 **base64,不是加密**;真實值別放進 git,改在外部建立。
- Secret 的消費方式跟 ConfigMap 完全一樣,外加 `kubernetes.io/tls` 型別餵給 Ingress 用的
  TLS 憑證。

---

**下一課:** Namespace 與資源管理 —— 隔離工作負載,並設定 CPU / 記憶體的 requests 與
limits。

[目錄](../README.md) · [物件參考](objects.zh.md) ·
[疑難排解](troubleshooting.zh.md)
