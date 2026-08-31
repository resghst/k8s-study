# Pods

> The smallest thing Kubernetes schedules, and the unit everything else is built
> on.

**Before this:** [kubectl](02-kubectl.md)

---

# Part 1 — Walkthrough

Do these in order. Expected output is shown after each command.

## Run your first Pod

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

`ContainerCreating` means the image is still being pulled. Wait a few seconds and
look again, this time with `-o wide`:

```bash
kubectl get pods -o wide
```

```
NAME   READY   STATUS    RESTARTS   AGE   IP           NODE     NOMINATED NODE   READINESS GATES
web    1/1     Running   0          12s   10.42.0.11   hankpc   <none>           <none>
```

`READY 1/1` and `Running` mean it is up. `-o wide` adds the **Pod IP**
(`10.42.0.11`) and the **node** it landed on (`hankpc`).

## See what happened to it

```bash
kubectl describe pod web
```

The important part is at the bottom:

```
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  12s   default-scheduler  Successfully assigned default/web to hankpc
  Normal  Pulled     12s   kubelet            Container image "nginx:alpine" already present on machine
  Normal  Created    12s   kubelet            Container created
  Normal  Started    12s   kubelet            Container started
```

That is the architecture lesson happening for real: **scheduler** picked a node,
then the **kubelet** pulled the image, created the container, and started it.

On a first run you would also see a `Pulling` event and a `Pulled` line reporting
the download time and image size.

## Read its logs

```bash
kubectl logs web
```

```
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
...
```

## Run a command inside it

```bash
kubectl exec web -- nginx -v
```

```
nginx version: nginx/1.31.4
```

Or get a shell:

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

## Pull out a single field

```bash
kubectl get pod web -o jsonpath='{.status.podIP}'
```

```
10.42.0.11
```

## Write your own manifest

Create [`playground/pod.yaml`](../playground/pod.yaml):

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

Note the difference in `LABELS`: your manifest set `app=hello`, while
`kubectl run` auto-generated `run=web`.

> **Tip** — you rarely have to write YAML from scratch:
> ```bash
> kubectl run hello --image=nginx:alpine --port=80 --dry-run=client -o yaml
> ```
> builds the object and prints it *without* sending it to the cluster.

## Watch self-healing, and find its limit

Kill the container's main process:

```bash
kubectl exec web -- kill 1
kubectl get pod web
```

```
NAME   READY   STATUS    RESTARTS     AGE
web    1/1     Running   1 (5s ago)   4m
```

`RESTARTS` went to 1 — the kubelet restarted the container **in place**. The Pod
is the same Pod, same IP.

Do it repeatedly and nginx keeps failing immediately, so the kubelet starts
backing off:

```
NAME   READY   STATUS             RESTARTS      AGE
web    0/1     CrashLoopBackOff   4 (30s ago)   6m
```

Now delete a Pod instead:

```bash
kubectl delete pod hello
kubectl get pods
```

```
pod "hello" deleted
NAME   READY   STATUS    RESTARTS   AGE
web    1/1     Running   1          6m
```

`hello` is gone and **nothing brings it back**. That is the key lesson: the
kubelet restarts *containers*, but nobody recreates a bare *Pod*. Lesson 4's
Deployment is what fixes that.

## Clean up

```bash
kubectl delete pod web --ignore-not-found
kubectl delete -f playground/pod.yaml --ignore-not-found
kubectl get pods
```

```
No resources found in default namespace.
```

---

# Part 2 — How it works

## Why a Pod, and not just a container

Most of the time you have one process to run, and a container is exactly that. But
sometimes two processes are genuinely one unit:

- a web server plus a helper that pulls fresh content into a shared directory
- an application plus a log or metrics shipper that reads its files
- a service plus a local proxy that terminates TLS

Those pairs need to **share a filesystem**, **reach each other on `localhost`**,
**always land on the same machine**, and **start and stop together**. A lone
container cannot express "we are a set".

The **Pod is that set** — the atom of scheduling and lifecycle:

- one **network identity**: a single IP and port space, so containers inside talk
  over `localhost`
- shared **volumes**
- **shared fate**: scheduled together onto one node; when the Pod goes, they all go

Kubernetes never schedules a container alone — it always wraps it in a Pod, even
when there is only one. That way every higher-level object (ReplicaSet,
Deployment, Service) has a single consistent unit to operate on.

**Most Pods hold exactly one container.** Multi-container Pods are for the
tightly-coupled helpers above — the sidecar pattern — not for unrelated services.

## Anatomy

```
Pod  (name, labels, one IP)
├── container A  ── image, command, ports, env, resources, probes
├── container B  ── (optional sidecar)
└── volumes      ── mounted into one or more containers
```

## The object's fields

```bash
kubectl get pod web -o yaml
```

| Field | Meaning |
| --- | --- |
| `apiVersion: v1`, `kind: Pod` | which object type this is |
| `metadata.name`, `metadata.labels` | name; labels — how Services and controllers select this Pod |
| `spec.containers[].image` | which image to run |
| `spec.containers[].ports[].containerPort` | informational: the port the process listens on |
| `spec.restartPolicy` | what to do when a container exits |
| `status.phase`, `status.podIP` | current phase, assigned IP |
| `status.conditions` | readiness detail |

`spec` is what you asked for; `status` is what the cluster reports — the same
split as every other object ([Architecture →
spec vs. status](01-architecture.md#spec-vs-status)).

## restartPolicy

Controls what the kubelet does when a container in the Pod exits.

| Value | Behaviour | Typical use |
| --- | --- | --- |
| `Always` *(default)* | restart whenever it exits, success or failure | long-running services |
| `OnFailure` | restart only on a non-zero exit | batch Jobs |
| `Never` | never restart | one-shot tasks you inspect afterwards |

Repeated fast failures produce `CrashLoopBackOff`: the kubelet waits longer before
each attempt (10s, 20s, 40s… capped at 5 minutes) so a broken container cannot
spin the node.

## Pod phases

| Phase | Meaning |
| --- | --- |
| `Pending` | accepted, but not all containers are running yet (scheduling, image pull) |
| `Running` | bound to a node, at least one container running |
| `Succeeded` | all containers exited 0 and will not restart |
| `Failed` | all containers terminated, at least one with a non-zero exit |
| `Unknown` | the node stopped reporting |

Note that `CrashLoopBackOff` and `ImagePullBackOff` are **not** phases — they are
container states shown in the `STATUS` column. See
[Troubleshooting](troubleshooting.md).

## Where self-healing stops

| What happened | Who fixes it | Result |
| --- | --- | --- |
| A container exits | kubelet, per `restartPolicy` | restarted in place; same Pod, same IP, `RESTARTS` increments |
| The Pod is deleted | nobody | gone for good |
| Its node fails | nobody | gone for good |

A bare Pod has no owner, so nothing recreates it. Giving it an owner — a
ReplicaSet, created and managed by a Deployment — is what turns "a container that
restarts" into "a service that survives".

---

# Part 3 — Command reference

### Listing and selecting

```bash
kubectl get pods                                     # basic list
kubectl get pods -o wide                             # + IP, node
kubectl get pods -w                                  # watch changes live
kubectl get pods -A                                  # all namespaces
kubectl get pods -l app=hello                        # by label
kubectl get pods --show-labels
kubectl get pods --field-selector status.phase=Running
kubectl get pod web -o jsonpath='{.status.podIP}'    # one field
kubectl get pod web -o yaml                          # the whole object
```

### Inspecting

```bash
kubectl describe pod web                  # fields + owner + Events
kubectl get events --sort-by=.lastTimestamp
```

### Logs

```bash
kubectl logs web
kubectl logs web -f                       # follow
kubectl logs web --previous               # the instance before the last restart
kubectl logs web -c <container>           # a specific container
kubectl logs web --since=15m --tail=100
kubectl logs -l app=hello --all-containers --prefix
```

### Getting in and out

```bash
kubectl exec web -- env                   # one command
kubectl exec -it web -- sh                # interactive shell
kubectl exec -it web -c <container> -- sh
kubectl port-forward pod/web 8080:80      # localhost:8080 -> container:80
kubectl cp web:/etc/nginx/nginx.conf ./nginx.conf
kubectl debug -it web --image=busybox --target=web   # image with no shell
```

### Changing and removing

```bash
kubectl label pod web env=test
kubectl annotate pod web owner=hank
kubectl top pod web                       # CPU/memory
kubectl delete pod web
kubectl delete pod web --grace-period=0 --force
kubectl delete -f playground/pod.yaml --ignore-not-found
```

### Discovering fields

```bash
kubectl explain pod.spec.containers
kubectl explain pod.spec.containers.livenessProbe
```

---

## Recap

- A Pod is the **atomic unit** of scheduling and lifecycle: shared IP, shared
  volumes, shared fate. One container is the common case.
- `kubectl run` for a throwaway Pod, `--dry-run=client -o yaml` to generate a
  manifest, `kubectl apply -f` for anything real.
- `describe` (Events) → `logs --previous` → `exec` is the standard debugging path.
- The kubelet restarts **containers**; nothing recreates a bare **Pod**.

---

**Next:** Deployments — giving your Pods an owner that recreates them.

[Contents](../README.md) · [Object reference](objects.md) ·
[Troubleshooting](troubleshooting.md)
