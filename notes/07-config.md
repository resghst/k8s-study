**English** · [中文](07-config.zh.md)

# ConfigMaps & Secrets

> Feed configuration and credentials into Pods at runtime, instead of baking them
> into the container image.

**Before this:** [Ingress](06-ingress.md)

Up to now configuration has been hardcoded — the nginx image's built-in page, edited
by hand with `kubectl exec`. Real apps need settings (database URLs, feature flags)
and credentials (passwords, API keys), and those must **not** live in the image:
changing one would mean rebuilding and redeploying, and any secret baked in leaks to
anyone who can pull the image. The fix is to decouple config from the image:

- **ConfigMap** — non-secret configuration (URLs, flags, config files).
- **Secret** — sensitive data (passwords, tokens, certificates).

The Pod reads them at startup. Change a setting = change the ConfigMap, no rebuild.

---

# Part 1 — Walkthrough

Do these in order.

## Create a ConfigMap and a Secret

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

## Look at them — a Secret is not encrypted

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

The ConfigMap stores plain text. The Secret's value looks scrambled — but that is
just **base64**, not encryption. Anyone with read access decodes it trivially:

```bash
kubectl get secret app-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d; echo
```

```
s3cr3t
```

A Secret differs from a ConfigMap by **intent and handling** — Kubernetes gates it
behind RBAC, keeps it out of logs, and can encrypt it at rest in etcd — but the
object itself does **not** encrypt your data. Never commit a Secret manifest with a
real value to git.

## Consume them in a Pod — two ways

[`playground/pod-config.yaml`](../playground/pod-config.yaml) injects values **as
environment variables** and mounts the ConfigMap **as files** at the same time.

```bash
kubectl apply -f playground/pod-config.yaml
```

As environment variables:

```bash
kubectl exec consumer -- printenv APP_COLOR DB_PASSWORD
```

```
blue
s3cr3t
```

`APP_COLOR` came from the ConfigMap, `DB_PASSWORD` from the Secret — and the Secret
was **decoded automatically** on the way into the container (you get `s3cr3t`, not
the base64).

As mounted files:

```bash
kubectl exec consumer -- sh -c 'ls /etc/config; cat /etc/config/APP_COLOR; cat /etc/config/APP_MODE'
```

```
APP_COLOR
APP_MODE
blue
production
```

Each ConfigMap key became a **file** under `/etc/config`, its value the file
contents. Same ConfigMap, consumed as variables or as files depending on what the
app expects.

## Clean up

```bash
kubectl delete pod consumer
kubectl delete configmap app-config
kubectl delete secret app-secret
```

---

# Part 2 — How it works

## Config lives outside the Pod, and is referenced by name

A ConfigMap or Secret is a standalone object in the API. A Pod does not contain the
values — it **references** the object by name (`configMapKeyRef`, `secretKeyRef`,
`configMap:` volume). This is the same decoupling pattern as a Service selecting
Pods by label: the config and the workload are separate objects, wired by reference.
One ConfigMap can feed many Pods; updating it updates the source for all of them.

## Two injection methods, and the update gotcha

| Method | How it appears | On ConfigMap/Secret change |
| --- | --- | --- |
| **Environment variable** (`valueFrom` / `envFrom`) | a process env var | **frozen** — the Pod must restart to see new values |
| **Volume mount** (`configMap:` / `secret:` volume) | files in a directory | **auto-updates** (kubelet re-syncs within ~a minute) |

Env vars are read once at container start and never revisited, so a changed value
needs `kubectl rollout restart deployment/<name>` to take effect. A mounted volume
is kept in sync by the kubelet, so an app that re-reads its config file picks up
changes without a restart. Choose the method by whether you need hot reloads.

## When to use which key style

- **`--from-literal=KEY=value`** (or `data:` key/value pairs) → best consumed as
  **env vars**, one setting per key.
- **`--from-file=app.conf`** (or a multi-line value) → the whole file becomes one
  key, best **mounted as a file** the app reads. See the `app.conf` key in
  [`playground/configmap.yaml`](../playground/configmap.yaml).

## stringData vs data

In a Secret manifest, `stringData` takes **plain text** and Kubernetes base64-encodes
it into `data` on apply — so you write `DB_PASSWORD: s3cr3t`, not the base64. Reading
the Secret back always shows the encoded `data`.
[`playground/secret.yaml`](../playground/secret.yaml) uses `stringData` (and is
flagged demo-only for exactly the git reason above).

## Keeping Secrets actually secret

Because base64 is not protection, real projects do **not** keep Secret values in the
repo. Common approaches: create them out-of-band (`kubectl create secret`), enable
**etcd encryption at rest**, or use an operator that injects them from a real vault
(sealed-secrets, external-secrets, or the cloud provider's secret manager). The Pod
spec still just references the Secret by name — only its origin changes.

---

# Part 3 — Command reference

### Create

```bash
# ConfigMap
kubectl create configmap app-config --from-literal=APP_COLOR=blue
kubectl create configmap app-config --from-file=app.conf          # whole file
kubectl create configmap app-config --from-env-file=app.env       # KEY=VALUE lines

# Secret
kubectl create secret generic app-secret --from-literal=DB_PASSWORD=s3cr3t
kubectl create secret tls my-tls --cert=tls.crt --key=tls.key      # for Ingress
```

### Inspect

```bash
kubectl get configmap app-config -o yaml
kubectl get secret app-secret -o yaml
kubectl get secret app-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d   # decode
kubectl describe configmap app-config
```

### Consume (manifest snippets)

```yaml
# one key as an env var
env:
  - name: APP_COLOR
    valueFrom:
      configMapKeyRef: { name: app-config, key: APP_COLOR }

# every key as env vars at once
envFrom:
  - configMapRef: { name: app-config }
  - secretRef:    { name: app-secret }

# mount as files
volumes:
  - name: config-vol
    configMap: { name: app-config }
volumeMounts:
  - { name: config-vol, mountPath: /etc/config, readOnly: true }
```

### Apply new values

```bash
kubectl apply -f playground/configmap.yaml     # update the ConfigMap
kubectl rollout restart deployment/web         # needed if consumed as env vars
```

### Discovering fields

```bash
kubectl explain pod.spec.containers.envFrom
kubectl explain pod.spec.volumes.configMap
```

---

## Recap

- **ConfigMap** (non-secret) and **Secret** (sensitive) decouple configuration from
  the image — change a setting without rebuilding.
- A Pod **references** them by name; one ConfigMap/Secret can feed many Pods.
- Inject them as **env vars** (frozen at start — restart to update) or as **mounted
  files** (auto-synced — good for hot reload).
- A Secret is **base64, not encrypted**; keep real values out of git and out-of-band.
- Secrets are consumed exactly like ConfigMaps, plus the `kubernetes.io/tls` type
  feeds the TLS certificate an Ingress uses.

---

**Next:** Namespaces & resource management — isolating workloads and setting CPU /
memory requests and limits.

[Contents](../README.md) · [Object reference](objects.md) ·
[Troubleshooting](troubleshooting.md)
