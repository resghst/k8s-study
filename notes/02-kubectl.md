**English** · [中文](02-kubectl.zh.md)

# kubectl

> The command-line client you will spend most of your time in. Worth learning
> properly rather than memorising a handful of lines.

**Before this:** [Architecture Overview](01-architecture.md)

---

# Part 1 — Walkthrough

## Confirm where you are pointed

```bash
kubectl config current-context
```

```
default
```

Do this **first** whenever output surprises you. Acting on the wrong cluster is a
common and expensive mistake.

## The command shape

Every command follows the same grammar:

```
kubectl <verb> <resource-type> [name] [flags]
```

```bash
kubectl get pod web -o yaml
#       verb resource name flag
```

Try each part:

```bash
kubectl get pods                       # verb + resource
kubectl get pods web                   # + a specific name
kubectl get pods web -o wide           # + a flag
```

```
NAME   READY   STATUS    RESTARTS   AGE   IP           NODE     NOMINATED NODE   READINESS GATES
web    1/1     Running   0          12s   10.42.0.11   hankpc   <none>           <none>
```

## Short names save typing

```bash
kubectl get po        # same as: kubectl get pods
kubectl get ns
```

```
NAME              STATUS   AGE
default           Active   65m
kube-system       Active   65m
kube-public       Active   65m
kube-node-lease   Active   65m
```

## Ask the cluster instead of searching the web

What resource types exist?

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

What fields does a Pod have?

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

What flags does a command take?

```bash
kubectl logs --help
```

Every `--help` ends with an **Examples** block. Read that before searching online.

## Extract just one value

```bash
kubectl get pod web -o jsonpath='{.status.podIP}'
```

```
10.42.0.11
```

## Build your own table

```bash
kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,IMAGE:.spec.containers[0].image
```

```
NAME   NODE     IMAGE
web    hankpc   nginx:alpine
```

## Generate a manifest instead of writing one

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

Nothing was created — `--dry-run=client` builds the object locally and prints it.

## Watch what kubectl really sends

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

# Part 2 — How it works

## kubectl is just a REST client

Every command turns into an HTTP request:

```
kubectl get pods
  -> GET https://127.0.0.1:6443/api/v1/namespaces/default/pods
```

kubectl has no special powers. It reads a **kubeconfig** to learn which cluster to
talk to and who to authenticate as, sends REST calls, and formats the JSON that
comes back. Anything kubectl does you could do with `curl` — kubectl just makes it
bearable.

That is why `-v=6` and `-v=8` are such good debugging tools: they show you the
exact request and response behind any command.

## kubeconfig and contexts

kubectl reads `~/.kube/config`, or whatever `KUBECONFIG` points at. A **context**
bundles three things:

- **cluster** — the API server URL and its CA certificate
- **user** — your credentials (certificate, token, or exec plugin)
- **namespace** — the default when you omit `-n`

Switching context switches all three at once. On k3s the context is called
`default` and lives in `/etc/rancher/k3s/k3s.yaml`.

## Why short names exist

Resource types accept singular, plural, and short forms — `pod`, `pods`, and `po`
are the same thing:

| Short | Full | Short | Full |
| --- | --- | --- | --- |
| `po` | pods | `cm` | configmaps |
| `deploy` | deployments | `pvc` | persistentvolumeclaims |
| `rs` | replicasets | `sa` | serviceaccounts |
| `svc` | services | `ing` | ingresses |
| `ns` | namespaces | `no` | nodes |

`kubectl api-resources` is the authoritative list for **your** cluster, including
short names added by custom resources.

## apply vs. create vs. replace

| Command | Behaviour |
| --- | --- |
| `apply` | create if absent, patch if present — **idempotent, use this** |
| `create` | fail if it already exists |
| `replace` | overwrite the whole object; fail if absent |
| `delete` | remove it |

`apply` is **declarative**: "make the cluster match this file". Run it again after
editing and only the difference is sent. It records what it applied in an
annotation, so it can also detect fields you *removed* — something `create` and
`replace` cannot do.

## The two dry-run modes

| Mode | What it does |
| --- | --- |
| `--dry-run=client` | builds the object locally, prints it, never contacts the cluster |
| `--dry-run=server` | sends it for full validation and defaulting, but does not persist |

Use `client` to generate a manifest skeleton; use `server` to catch schema errors
before a real apply.

## describe is not `get -o yaml`

`get -o yaml` dumps one object. `describe` **joins related information**: the
object's fields, its owner, and the recent **Events** about it. When something is
broken, the Events section names the reason. That is why the debugging path always
starts with `describe`.

## Namespaces

Most resources live in a namespace; without `-n`, kubectl uses the context's
default. Cluster-scoped objects (nodes, namespaces, PersistentVolumes,
ClusterRoles) ignore `-n` entirely — `kubectl api-resources --namespaced=false`
lists them.

---

# Part 3 — Command reference

### Context and cluster

```bash
kubectl config current-context
kubectl config get-contexts              # * marks the active one
kubectl config view --minify             # active config, secrets redacted
kubectl config use-context <name>
kubectl config set-context --current --namespace=dev
kubectl cluster-info
kubectl version
```

### Discovery and help

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

### Listing

```bash
kubectl get pods
kubectl get pods -o wide
kubectl get pods -A                      # all namespaces
kubectl get pods -n kube-system
kubectl get pods -w                      # watch until Ctrl+C
kubectl get all                          # common workload types at once
```

### Output formats

```bash
kubectl get pods -o yaml
kubectl get pods -o json
kubectl get pods -o name                            # "pod/web"
kubectl get pod web -o jsonpath='{.status.podIP}'
kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName
```

### Filtering

```bash
kubectl get pods -l app=hello                       # label selector
kubectl get pods -l 'env in (dev,test)'             # set-based
kubectl get pods --show-labels
kubectl get pods --field-selector status.phase=Running
kubectl get pods --sort-by=.metadata.creationTimestamp
```

### Inspecting

```bash
kubectl describe pod web
kubectl describe node <name>
kubectl get events --sort-by=.lastTimestamp
kubectl get events -A --sort-by=.lastTimestamp | tail -30
kubectl get events --field-selector type=Warning
```

### Creating and changing

```bash
kubectl apply -f pod.yaml
kubectl apply -f ./manifests/
kubectl apply -k ./overlays/dev                     # kustomize
kubectl apply -f pod.yaml --dry-run=server          # validate only
kubectl diff -f pod.yaml                            # preview the change

kubectl run web --image=nginx:alpine --dry-run=client -o yaml
kubectl create deployment web --image=nginx:alpine --dry-run=client -o yaml

kubectl edit pod web
kubectl scale deploy web --replicas=3
kubectl set image deploy/web web=nginx:1.27
kubectl label pod web env=test
kubectl annotate pod web owner=hank
kubectl patch pod web -p '{"metadata":{"labels":{"tier":"front"}}}'
```

### Deleting

```bash
kubectl delete pod web
kubectl delete -f pod.yaml
kubectl delete -f pod.yaml --ignore-not-found
kubectl delete pods -l app=hello
kubectl delete pod web --grace-period=0 --force
```

### Debugging

```bash
kubectl logs web
kubectl logs web -f
kubectl logs web --previous                         # before the last restart
kubectl logs web -c <container>
kubectl logs web --since=15m --tail=100
kubectl logs -l app=hello --all-containers --prefix

kubectl exec web -- env
kubectl exec -it web -- sh
kubectl debug -it web --image=busybox --target=web  # image with no shell

kubectl port-forward pod/web 8080:80
kubectl port-forward svc/web 8080:80
kubectl cp web:/etc/nginx/nginx.conf ./nginx.conf

kubectl top nodes
kubectl top pods
```

### Flags that work almost everywhere

| Flag | Meaning |
| --- | --- |
| `-n <ns>` / `--namespace` | target namespace |
| `-A` / `--all-namespaces` | every namespace |
| `-o <format>` | output format |
| `-l <selector>` | filter by label |
| `-f <file\|dir\|url>` | read objects from a manifest |
| `-w` | watch for changes |
| `--dry-run=client\|server` | build/validate without persisting |
| `-v=6` … `-v=8` | show the underlying API calls |
| `--context` / `--kubeconfig` | override the target cluster for one command |

### Making it comfortable

```bash
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
```

---

## Recap

- kubectl is a thin REST client; `-v=8` shows exactly what it sent.
- Learn `verb resource name flags` once and every command follows.
- `--help`, `api-resources`, and `explain` answer most questions **from your own
  cluster**.
- `apply` for anything real, `--dry-run=client -o yaml` to generate a manifest,
  `diff` to preview.
- `describe` (Events) → `logs --previous` → `exec` is the standard debugging path.
- Always know your **context** before trusting what you see.

---

**Next:** [Pods](03-pods.md) — the first object you will actually create.

[Contents](../README.md) · [Troubleshooting](troubleshooting.md)
