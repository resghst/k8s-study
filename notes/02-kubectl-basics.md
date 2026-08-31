# kubectl 常用指令

## 查詢資源

```bash
kubectl get pods
kubectl get deployments
kubectl describe pod <pod-name>
```

## 套用設定

```bash
kubectl apply -f deployment.yaml
```

## 除錯

```bash
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- /bin/sh
```
