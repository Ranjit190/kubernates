# Kubernetes Practice Notes

Hands-on notes for learning Kubernetes locally with **kind** (Kubernetes in Docker).
Commands and concepts are grouped by topic, with the matching manifests in this repo
referenced where relevant.

---

## Table of Contents
1. [Local Setup (Docker + kind)](#1-local-setup-docker--kind)
2. [Namespaces](#2-namespaces)
3. [Pods](#3-pods)
4. [Workloads (Deployment, ReplicaSet, DaemonSet, StatefulSet)](#4-workloads)
5. [Updates: Rolling Update vs Rollback](#5-updates-rolling-update-vs-rollback)
6. [Scaling & Scheduling (Resource limits, HPA, VPA, metrics-server)](#6-scaling--scheduling)
7. [Networking (Service, port-forward, Ingress)](#7-networking)
8. [Storage (PersistentVolume & PersistentVolumeClaim)](#8-storage)
9. [Configuration (ConfigMap & Secret)](#9-configuration)
10. [Jobs](#10-jobs)
11. [Debugging Cheat Sheet](#11-debugging-cheat-sheet)
12. [Manifests in This Repo](#12-manifests-in-this-repo)

---

## 1. Local Setup (Docker + kind)

### Install the tools
You need three things — and note **`kubectl` is separate**: kind creates the cluster, but
`kubectl` is the CLI you talk to it with (kind does *not* bundle it).

```bash
# 1. Docker   -> https://docs.docker.com/engine/install/
# 2. kind     -> https://kind.sigs.k8s.io/docs/user/quick-start/
# 3. kubectl  (NOT installed by kind):
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

On Linux, let your user run Docker without `sudo`, then refresh the group in the current
shell (it's `usermod`, not `usermode`):

```bash
sudo usermod -aG docker $USER && newgrp docker
```

### Create the cluster *with a config* (so Ingress works)
A bare `kind create cluster` can't receive traffic on ports 80/443, and the ingress-nginx
controller only schedules onto a node labelled `ingress-ready=true`. Both must be set **at
creation time** via a config file — you can't add them later. Save this as `kind-config.yml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
```

```bash
kind create cluster --name practice --config kind-config.yml
kubectl cluster-info --context kind-practice
kubectl get nodes
```

- `extraPortMappings` forwards your host's `localhost:80/443` into the node, so
  `curl localhost/apache` reaches the Ingress.
- `node-labels: ingress-ready=true` lets the ingress controller pod schedule — without it
  the controller stays `Pending` forever.

### Install cluster add-ons
These aren't part of kind; install them once the cluster is up:

```bash
# Ingress controller (kind-specific manifest — needed by apache/ingress.yml)
kubectl apply -f https://kind.sigs.k8s.io/examples/ingress/deploy-ingress-nginx.yaml
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s

# metrics-server (needed by HPA/VPA + `kubectl top`) — see Section 6 for the kind TLS patch
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

> Hitting `too many open files` on a larger cluster? That's a host sysctl limit, not
> Kubernetes:
> ```bash
> sudo sysctl fs.inotify.max_user_watches=524288
> sudo sysctl fs.inotify.max_user_instances=512
> ```

---

## 2. Namespaces

A namespace is a logical partition inside the cluster.

```bash
# Imperative
kubectl create namespace nginx

# Declarative (preferred) — see apache/namespace.yml
kubectl apply -f apache/namespace.yml
```

Almost every command below takes `-n <namespace>` to target a namespace.

---

## 3. Pods

The smallest deployable unit — one or more containers that share network/storage.

```bash
# Run a one-off pod imperatively
kubectl run nginx --image=nginx -n nginx

# Declarative is preferred for anything you keep
kubectl apply -f pod.yml
```

You rarely create bare Pods in production — you let a **Deployment** manage them so they
self-heal and scale.

---

## 4. Workloads

| Kind | What it does | Can roll out updates? |
|------|--------------|-----------------------|
| **Pod** | A single instance; no self-healing on its own | — |
| **ReplicaSet** | Keeps N identical pods running | ❌ No rollout/versioning |
| **Deployment** | A ReplicaSet **plus** rolling updates & rollback | ✅ Yes |
| **DaemonSet** | Ensures **one pod on every node** (e.g. log/metrics agents) | ✅ Yes |
| **StatefulSet** | For stateful apps (databases); stable network IDs & storage | ✅ Yes |

**Deployment** — the everyday workload. See `apache/deployment.yml`.

```bash
kubectl apply -f apache/deployment.yml
kubectl scale deployment/apache-deployment -n apache --replicas=5
```

**ReplicaSet** — almost identical YAML to a Deployment; only `kind:` and a few names
differ. A Deployment manages ReplicaSets for you, so you seldom write one directly.
The trade-off: a ReplicaSet **cannot do rollouts/rollbacks**.

**DaemonSet** — guarantees at least one pod per node. Good for node-level agents.

**StatefulSet** — for workloads that need a stable identity (databases, queues). When a
pod crashes it is recreated with the **same name and the same persistent volume**, unlike
a Deployment where pods get random names. Usually paired with a **headless Service**.

---

## 5. Updates: Rolling Update vs Rollback

These are two different things — your notes lumped them together.

**Rolling update** = the default Deployment strategy. New pods come up before old ones go
down, so there is **no downtime**. Triggered by changing the spec, e.g. the image:

```bash
kubectl set image deployment/nginx-deployment -n nginx nginx=nginx:1.27.3
kubectl rollout status deployment/nginx-deployment -n nginx
```

**Rollback** = revert to a previous revision if an update went bad:

```bash
kubectl rollout history deployment/nginx-deployment -n nginx
kubectl rollout undo deployment/nginx-deployment -n nginx          # previous revision
kubectl rollout undo deployment/nginx-deployment -n nginx --to-revision=2
```

---

## 6. Scaling & Scheduling

### Resource requests & limits
Set CPU/memory **requests** (guaranteed) and **limits** (ceiling) per container in the
Deployment. Requests are also required for HPA to work. See `apache/deployment.yml`:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "200m"
    memory: "256Mi"
```

> A **ResourceQuota** (a separate object) caps total CPU/memory/object-count for a whole
> namespace — useful for multi-team clusters.

### Horizontal Pod Autoscaler (HPA)
HPA scales replica count up/down based on observed load (e.g. CPU %). You define min/max
replicas and a target utilization. See `apache/hpa.yml`.

```bash
kubectl apply -f apache/hpa.yml
kubectl get hpa -n apache
```

You can also tune **how fast** it scales using the `behavior` block
(`stabilizationWindowSeconds`, scale-up/scale-down policies) — already demonstrated in
`apache/hpa.yml`.

### metrics-server (required for HPA + `kubectl top`)
A fresh kind cluster has no metrics, so HPA can't read CPU. Install metrics-server:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

> On **kind**, metrics-server often fails TLS verification against the kubelet. If it
> stays `0/1 Ready`, patch it to skip kubelet TLS (fine for local practice only):
> ```bash
> kubectl patch deployment metrics-server -n kube-system --type=json \
>   -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
> ```

Confirm it works:

```bash
kubectl top nodes
kubectl top pods -n <namespace>
```

> Note: `kubectl top nodes` is cluster-scoped (no `-n`). Use `-n` only with `top pods`.

### Vertical Pod Autoscaler (VPA)
Where HPA scales **out** (more pods), VPA scales **up** — it right-sizes a single pod's
CPU/memory **requests & limits** based on real usage, so you neither over-provision nor get
throttled/OOM-killed. See `apache/vpa.yml`.

Unlike HPA, VPA is **not** built into the cluster — install it from the autoscaler repo:

```bash
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh        # installs the recommender, updater, and admission controller
# ./hack/vpa-down.sh    # to uninstall later
```

`updateMode` controls how aggressive it is:
- `"Off"` — only *recommends* values (read them with `kubectl describe vpa`); changes nothing.
- `"Initial"` — applies resources only when a pod is first created.
- `"Auto"` / `"Recreate"` — evicts and recreates running pods to apply new resources (this
  is what `apache/vpa.yml` uses).

```bash
kubectl apply -f apache/vpa.yml
kubectl get vpa -n apache
kubectl describe vpa apache-vpa -n apache    # view the recommendation
```

> ⚠️ **Don't point HPA and VPA at the same metric** (e.g. both on CPU) for one workload —
> they fight each other. In this repo, `apache/hpa.yml` and `apache/vpa.yml` *both* target
> `apache-deployment` on CPU, so apply only one at a time. Valid combos: VPA for CPU/memory
> **or** HPA for CPU; or HPA on custom metrics **plus** VPA on CPU/memory. VPA also needs
> **metrics-server**.

---

## 7. Networking

### Service
Anything running in the cluster is reached through a **Service**, which gives a stable
virtual IP/DNS name in front of a set of pods (matched by label selector).
See `apache/service.yml`. Common types: `ClusterIP` (internal, default),
`NodePort`, `LoadBalancer`.

### Port-forward (quick external access for testing)
```bash
kubectl port-forward service/nginx-service -n nginx 80:80 --address=0.0.0.0
```

### Ingress (HTTP routing by host/path)
An Ingress exposes multiple services under one entry point, routing by path or host —
e.g. `/apache` → apache, `/nginx` → nginx. See `apache/ingress.yml`.

Key points:
- Ingress is **namespaced**: it can only route to Services in **its own namespace**.
- It needs an **ingress controller** installed (e.g. ingress-nginx).
- Path prefixes usually need a `rewrite-target` annotation so the backend (which serves
  at `/`) receives the stripped path.

```bash
kubectl apply -f apache/ingress.yml
kubectl get ingress -n apache
```

---

## 8. Storage

Pods are ephemeral — for data that must survive restarts you need volumes.

- **PersistentVolume (PV)** — a piece of storage in the cluster. **Cluster-scoped**
  (no namespace). See `presistanceVolume.yml` and `mysql/persistentVolume.yml`.
- **PersistentVolumeClaim (PVC)** — a request for storage by a pod. **Namespaced**. It
  binds to a PV with a matching `storageClassName`, access mode, and size. See
  `mysql/persistentVolumeClaim.yml`.

The pod mounts the PVC as a volume:

```yaml
volumeMounts:
- name: mysql-storage
  mountPath: /var/lib/mysql
volumes:
- name: mysql-storage
  persistentVolumeClaim:
    claimName: mysql-pvc
```

```bash
kubectl get pv
kubectl get pvc -n mysql
```

---

## 9. Configuration

- **ConfigMap** — non-sensitive key/value config injected as env vars or mounted files.
- **Secret** — base64-encoded sensitive values (passwords, tokens). Reference it from a
  container with `secretKeyRef`. See `mysql/secret.yml` and how
  `mysql/deployment.yml` consumes it.

```bash
kubectl get secret -n mysql
kubectl create secret generic db-pass --from-literal=password=changeme -n mysql   # imperative example
```

> Never commit real credentials. Secrets are only base64-encoded, **not encrypted**.

---

## 10. Jobs

A **Job** runs a pod to completion (batch work), then stops. A **CronJob** runs Jobs on a
schedule. See `job.yml`.

```bash
kubectl apply -f job.yml
kubectl get jobs -n <namespace>
kubectl logs job/demo-job-1 -n <namespace>
```

Key fields: `completions`, `parallelism`, `backoffLimit`, and `restartPolicy: Never`.

---

## 11. Debugging Cheat Sheet

```bash
# Open a shell inside a running pod
kubectl exec -it nginx-pod -n nginx -- bash       # use -- sh if bash isn't present

# Inspect a pod's events, state, and why it won't start
kubectl describe pod/nginx-pod -n nginx

# Logs (add -f to follow, --previous for a crashed container)
kubectl logs nginx-pod -n nginx -f

# Everything in a namespace at a glance
kubectl get all -n nginx

# Watch pods change state live
kubectl get pods -n nginx -w
```

> The shell separator is `--` (two hyphens) and the shell is `bash`/`sh` — your note's
> `– base` was an en-dash + typo.

---

## 12. Manifests in This Repo

| Path | Kind | Namespace | Notes |
|------|------|-----------|-------|
| `apache/namespace.yml` | Namespace | — | `apache` namespace |
| `apache/deployment.yml` | Deployment | apache | httpd, with resource requests/limits |
| `apache/service.yml` | Service | apache | ClusterIP :80 |
| `apache/hpa.yml` | HorizontalPodAutoscaler | apache | CPU-based, min 1 / max 5 |
| `apache/vpa.yml` | VerticalPodAutoscaler | apache | right-sizes CPU/mem, `updateMode: Auto` |
| `apache/ingress.yml` | Ingress | apache | `/apache` & `/nginx` routing |
| `mysql/namespace.yml` | Namespace | — | `mysql` namespace |
| `mysql/secret.yml` | Secret | mysql | root password (placeholder) |
| `mysql/persistentVolume.yml` | PersistentVolume | — | cluster-scoped, 1Gi hostPath |
| `mysql/persistentVolumeClaim.yml` | PersistentVolumeClaim | mysql | binds to the PV |
| `mysql/deployment.yml` | Deployment | mysql | mysql:8.0, mounts PVC, reads Secret |
| `mysql/service.yml` | Service | mysql | ClusterIP :3306 |
| `job.yml` | Job | nginx | busybox run-to-completion demo |
| `presistanceVolume.yml` | PersistentVolume | — | standalone PV example |
| `service.yml` | Service | default | standalone nginx Service example |

### Typical apply order
```bash
kubectl apply -f apache/namespace.yml
kubectl apply -f apache/deployment.yml
kubectl apply -f apache/service.yml
kubectl apply -f apache/hpa.yml          # needs metrics-server
# kubectl apply -f apache/vpa.yml        # needs VPA installed (vpa-up.sh); don't run alongside hpa.yml
kubectl apply -f apache/ingress.yml      # needs an ingress controller
```
