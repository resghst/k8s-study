# Troubleshooting

## A Pod won't start

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> --previous   # logs from the last crashed container
```

Common STATUS values:

| STATUS | Likely cause |
| --- | --- |
| `ImagePullBackOff` | Wrong image name or tag; private registry with no imagePullSecret |
| `CrashLoopBackOff` | Container exits right after starting — check `logs --previous` |
| `Pending` | Not enough resources or no matching node — check the Events in `describe` |
| `CreateContainerConfigError` | A referenced ConfigMap or Secret does not exist |

## A Service is unreachable

```bash
kubectl get endpoints <service-name>   # no endpoints means the selector matches no Pods
kubectl get pods --selector=<key>=<value>
```

First confirm the Service `selector` matches the Pod labels, then confirm
`targetPort` matches the port the container actually listens on.

## Node status

```bash
kubectl get nodes
kubectl describe node <node-name>   # check Conditions and Taints
```
