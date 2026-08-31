# Pods

Lesson 2. The Pod is the smallest thing Kubernetes schedules.

## What a Pod is

- You cannot schedule a bare "container" — only a **Pod**.
- A Pod holds **one or more containers** that share:
  - a **network namespace** — one Pod IP, one port space, so containers in the
    same Pod reach each other over `localhost`
  - access to the same **volumes**
- Multiple containers in one Pod is the **sidecar** pattern (a main container plus
  a helper — log shipper, proxy, ...). Single-container Pods are the common case.
- In practice you **rarely write a Pod directly**. A Deployment (Lesson 3)
  creates and manages them for you. You still need to understand Pods first.

## Run one

```bash
kubectl run web --image=nginx:alpine
kubectl get pods
kubectl get pods -o wide          # also shows Pod IP and the node it landed on
```

## Observe it

```bash
kubectl describe pod web
```

Read the **Events** section at the bottom — it is the Lesson 1 flow happening for
real: `Scheduled` → `Pulling` → `Pulled` → `Created` → `Started`.

```bash
kubectl logs web
kubectl exec -it web -- sh        # a shell inside the container; `exit` to leave
```

## The object

```bash
kubectl get pod web -o yaml
```

| Field | Meaning |
| --- | --- |
| `apiVersion: v1`, `kind: Pod` | which object type this is |
| `metadata.name`, `metadata.labels` | name; labels (used later by Services / controllers to select this Pod) |
| `spec.containers[].image` | which image to run |
| `spec.containers[].ports[].containerPort` | informational: the port the container listens on |
| `status.phase`, `status.podIP` | current phase, assigned IP |

`spec` is what you asked for; `status` is what the cluster reports. That split is
common to every Kubernetes object.

## Write a manifest

[`playground/pod.yaml`](../playground/pod.yaml):

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
kubectl get pods --show-labels
```

## Self-healing and its boundary

```bash
# kill the container's main process -> kubelet restarts the container in place
kubectl exec web -- kill 1
kubectl get pod web -w            # RESTARTS increments; Ctrl+C to stop watching

# delete the Pod -> it is gone; a bare Pod is NOT recreated
kubectl delete pod hello
kubectl get pods
```

- A **container** that exits is restarted by the kubelet according to the Pod's
  `restartPolicy`. Restarting the same failing container quickly leads to
  `CrashLoopBackOff` / `BackOff`.
- A **Pod** that is deleted, or that was on a node which failed, is **not**
  recreated on its own. Something has to own it — that is what a **ReplicaSet /
  Deployment** does (Lesson 3).

### restartPolicy

| Value | Behaviour | Typical use |
| --- | --- | --- |
| `Always` (default) | restart the container whenever it exits | long-running services |
| `OnFailure` | restart only on non-zero exit | batch Jobs |
| `Never` | never restart | one-shot tasks where you inspect the result |

### Pod phases

| Phase | Meaning |
| --- | --- |
| `Pending` | accepted, not all containers running yet (scheduling, image pull) |
| `Running` | bound to a node, at least one container running |
| `Succeeded` | all containers exited 0, will not restart |
| `Failed` | all containers terminated, at least one non-zero |
| `Unknown` | node stopped reporting |

## Clean up

```bash
kubectl delete pod web
kubectl delete -f playground/pod.yaml --ignore-not-found
```

## Takeaways

- Pod = smallest schedulable unit; containers in it share network and volumes.
- `kubectl run` for a throwaway Pod; a manifest + `kubectl apply` for anything
  real.
- `describe` (Events), `logs`, `exec` are the three inspection tools.
- The kubelet restarts **containers**; it does not recreate **Pods**. Use a
  Deployment when you need the Pod to come back.
