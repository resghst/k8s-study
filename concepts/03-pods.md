# 03 Pod(概念)

> 一句話:**Pod 是 K8s 排程的最小單位——通常就是「一個容器 + 它的執行環境」。**

## 解決什麼問題

K8s 不直接管「容器」,它管「Pod」。Pod 是一層薄薄的包裝,讓一個(偶爾多個)容器共用同一個 IP、同一組儲存、同生共死。

## 心智模型

- **一個 Pod 通常 = 一個容器。**(多容器只在「緊密相依、要共用網路/檔案」時才用,例如 sidecar。)
- 同一個 Pod 裡的容器:**共用 IP、共用 volume、一起被排程、一起生死。**
- **Pod 是「用完可丟」的**:它拿到的 IP 隨時會變,不該把它當固定的東西。

## 一件事看穿全局

自癒的界線在這裡:**kubelet 會幫你重啟掛掉的「容器」,但沒有人會重建被刪掉的「Pod」。** 你手動 `kubectl delete pod`,它就真的沒了——這正是下一課 Deployment 要補的洞。

## 最小例子

```yaml
apiVersion: v1
kind: Pod
metadata: { name: web }
spec:
  containers:
    - name: web
      image: nginx:alpine
```

## 一句話記住

**Pod 會跑,但很脆弱、會換 IP、被刪不會回來。** 所以實務上幾乎不會裸開 Pod。

---
🔧 動手做:[notes/03-pods.md](../notes/03-pods.md)
← [02 kubectl](02-kubectl.md)｜→ [04 Deployment](04-deployments.md)
