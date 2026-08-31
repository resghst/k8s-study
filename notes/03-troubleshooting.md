# 除錯速查

## Pod 起不來

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> --previous   # 看上一個崩潰的 container
```

常見 STATUS 對照:

| STATUS | 可能原因 |
| --- | --- |
| `ImagePullBackOff` | image 名稱或 tag 錯、私有 registry 未設 imagePullSecret |
| `CrashLoopBackOff` | container 啟動後隨即結束,看 `logs --previous` |
| `Pending` | 資源不足或無符合的節點,看 `describe` 的 Events |
| `CreateContainerConfigError` | 參照的 ConfigMap / Secret 不存在 |

## Service 連不到

```bash
kubectl get endpoints <service-name>   # 沒有 endpoint 代表 selector 對不到 Pod
kubectl get pods --selector=<key>=<value>
```

先確認 Service 的 `selector` 與 Pod 的 label 一致,再確認 `targetPort` 對到 container 實際監聽的 port。

## 節點狀態

```bash
kubectl get nodes
kubectl describe node <node-name>   # 看 Conditions 與 Taints
```
