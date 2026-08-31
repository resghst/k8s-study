# kubectl Basics

## Inspect resources

```bash
kubectl get pods
kubectl get deployments
kubectl describe pod <pod-name>
```

## Apply configuration

```bash
kubectl apply -f deployment.yaml
```

## Debug

```bash
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- /bin/sh
```
