# Pods

Lesson 2. The Pod is the smallest thing Kubernetes schedules and the unit
everything else is built on.

## 1. Why a Pod, and not just a container

Most of the time you have one process you want to run, and a container is exactly
that. But sometimes two processes are genuinely one unit:

- a web server and a helper that pulls fresh content into a shared directory
- an application and a log/metrics shipper that reads its files
- a service and a local proxy that terminates TLS or talks to a mesh

Those pairs need to **share a filesystem**, **reach each other over
`localhost`**, **always land on the same machine**, and **start and stop
together**. A lone container cannot express "we are a set".

The **Pod is that set**. It is the atom of scheduling and lifecycle:

- one **network identity** — a single IP, one port space, so containers inside
  reach each other on `localhost`
- shared **volumes**
- **shared fate** — scheduled together onto one node, and when the Pod is
  removed, all its containers go

Kubernetes never schedules a container on its own; it always wraps it in a Pod,
even when there is just one. That way every higher-level thing — ReplicaSets,
Deployments, Services — has a single consistent unit to operate on.

**In practice, most Pods hold exactly one container.** Multi-container Pods are
for the tightly-coupled helpers above (the sidecar / adapter / ambassador
patterns), not for unrelated services.

## 2. Anatomy

```
Pod  (name, labels, one IP)
├── container A  ── image, command, ports, env, resources, probes
├── container B  ── (optional sidecar)
└── volumes      ── mounted into one or more of the containers
```

You also rarely create Pods directly — a Deployment (Lesson 3) creates and
replaces them for you. Understanding the Pod first is what makes Deployments make
sense.

## 3. Run one

```bash
kubectl run web --image=nginx:alpine
kubectl get pods
kubectl get pods -o wide          # adds Pod IP and the node it landed on
```

`kubectl run` is the quick way to get a throwaway Pod. For anything you want to
keep or version-control, write a manifest (§5).

## 4. Inspect it

Three tools cover almost everything:

```bash
kubectl describe pod web
```

Scroll to **Events** at the bottom — this is the Lesson 1 flow happening for
real:

```
Scheduled    -> the scheduler picked a node
Pulling      -> kubelet told the runtime to fetch the image
Pulled       -> image is local (with size and how long it took)
Created      -> container object created
Started      -> process is running
```

If a Pod is stuck, its failure almost always shows here (image pull errors,
volume mount failures, failing probes).

```bash
kubectl logs web                 # stdout/stderr of the (only) container
kubectl logs web -f              # follow, like tail -f
kubectl logs web --previous      # logs from the container instance before the last restart
kubectl logs web --since=15m --tail=100

kubectl exec web -- ls /etc/nginx        # run one command inside
kubectl exec -it web -- sh               # interactive shell; `exit` to leave
```

## 5. The object

```bash
kubectl get pod web -o yaml
```

| Field | Meaning |
| --- | --- |
| `apiVersion: v1`, `kind: Pod` | which object type this is |
| `metadata.name`, `metadata.labels` | name; labels (Services and controllers select Pods by label) |
| `spec.containers[].image` | which image to run |
| `spec.containers[].ports[].containerPort` | informational — the port the process listens on |
| `spec.restartPolicy` | what to do when a container exits (§7) |
| `status.phase`, `status.podIP`, `status.conditions` | current phase, assigned IP, readiness detail |

`spec` is what you asked for; `status` is what the cluster reports. That split is
common to every Kubernetes object (see
[Architecture §4](00-architecture.md#4-spec-vs-status)).

## 6. Write a manifest

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

### Generate the YAML instead of writing it by hand

```bash
kubectl run hello --image=nginx:alpine --port=80 \
  --dry-run=client -o yaml > playground/pod.yaml
```

`--dry-run=client` builds the object and prints it without sending it to the
cluster. This is the normal way to get a correct skeleton to edit.

## 7. Self-healing, and where it stops

```bash
# kill the container's main process -> the kubelet restarts the container in place
kubectl exec web -- kill 1
kubectl get pod web -w            # RESTARTS increments; Ctrl+C to stop watching

# delete the Pod -> it is gone; a bare Pod is NOT recreated
kubectl delete pod hello
kubectl get pods
```

- A **container** that exits is restarted by the kubelet per the Pod's
  `restartPolicy`. Fast repeated failures become `CrashLoopBackOff` / `BackOff`
  (the kubelet waits longer between each attempt).
- A **Pod** that is deleted, or that lived on a node which failed, is **not**
  recreated on its own. Something has to own it — that is a **ReplicaSet /
  Deployment** (Lesson 3).

### restartPolicy

| Value | Behaviour | Typical use |
| --- | --- | --- |
| `Always` (default) | restart the container whenever it exits | long-running services |
| `OnFailure` | restart only on a non-zero exit | batch Jobs |
| `Never` | never restart | one-shot tasks where you inspect the result |

### Pod phases

| Phase | Meaning |
| --- | --- |
| `Pending` | accepted, not all containers running yet (scheduling, image pull) |
| `Running` | bound to a node, at least one container running |
| `Succeeded` | all containers exited 0, will not restart |
| `Failed` | all containers terminated, at least one non-zero |
| `Unknown` | the node stopped reporting |

## 8. More commands you will use with Pods

### Selecting and formatting

```bash
kubectl get pods -w                                  # watch changes live
kubectl get pods -o wide                             # + IP, node
kubectl get pods -o name                             # just "pod/web"
kubectl get pod web -o jsonpath='{.status.podIP}'    # pull one field
kubectl get pods -l app=hello                        # label selector
kubectl get pods --field-selector status.phase=Running
kubectl get pods -A                                  # every namespace
```

### Logs and exec (multi-container aware)

```bash
kubectl logs web -c <container>        # pick a container in a multi-container Pod
kubectl logs -l app=hello --all-containers --tail=20   # across all Pods with a label
kubectl exec -it web -c <container> -- sh
kubectl exec web -- env
```

### Getting in and out

```bash
kubectl port-forward pod/web 8080:80      # localhost:8080 -> container:80
kubectl cp web:/etc/nginx/nginx.conf ./nginx.conf
kubectl cp ./index.html web:/usr/share/nginx/html/index.html
```

### Metadata and lifecycle

```bash
kubectl label pod web env=test
kubectl annotate pod web owner=hank
kubectl top pod web                       # CPU/memory (needs metrics-server; k3s has it)
kubectl delete pod web
kubectl delete pod web --grace-period=0 --force     # don't wait for graceful shutdown
kubectl delete -f playground/pod.yaml --ignore-not-found
```

### Discovering fields

```bash
kubectl explain pod.spec.containers.resources
kubectl explain pod.spec.containers.livenessProbe
```

## 9. Clean up

```bash
kubectl delete pod web --ignore-not-found
kubectl delete -f playground/pod.yaml --ignore-not-found
kubectl get pods            # back to "No resources found"
```

## 10. Takeaways

- A Pod is the **atomic unit** of scheduling and lifecycle: shared IP, shared
  volumes, shared fate. One container is the common case; multiple means
  tightly-coupled helpers.
- `kubectl run` for a throwaway Pod; `--dry-run=client -o yaml` to generate a
  manifest; `kubectl apply -f` for anything real.
- `describe` (Events), `logs`, `exec` are the three inspection tools; `-o
  jsonpath`, `-l`, `-w`, `port-forward` are the daily extras.
- The kubelet restarts **containers**; it does not recreate **Pods**. Use a
  Deployment when you need the Pod to come back.
