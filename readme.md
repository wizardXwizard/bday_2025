Perfect. I’ll write this like a **field manual** you can follow end-to-end without coming back, with **clear checkpoints**, **no branching**, and **no hand-waving**.

This is **single-source-of-truth documentation** for *your* setup.

---

# 📘 Elite Local GitOps Lab

**Phone = k3s workload cluster**
**Laptop = future control plane (KIND + ArgoCD)**

You will execute this in **two stages**:

* **Stage 1 (NOW)** → k3s on phone, fully working, production-style
* **Stage 2 (LEVEL UP)** → KIND + ArgoCD on laptop, multi-cluster GitOps

Nothing is wasted. Stage 2 builds directly on Stage 1.

---

# STAGE 1 — k3s on Phone (FOUNDATION)

## Goal of Stage 1

* Phone runs a **real Kubernetes cluster**
* You understand **why each piece exists**
* Laptop can already talk to it via `kubectl`
* No GitOps yet (don’t rush)

---

## 1️⃣ Phone prerequisites (must be true)

✔ Rooted Android
✔ Termux installed
✔ Stable Wi-Fi (phone + laptop on same network)
✔ At least **4 GB RAM free**
✔ 6–8 GB disk free

If these aren’t true → stop.

---

## 2️⃣ Install Ubuntu in Termux (clean base)

Inside **Termux**:

```bash
pkg update && pkg upgrade
pkg install proot-distro
proot-distro install ubuntu
proot-distro login ubuntu
```

From now on:

> **All commands happen inside Ubuntu unless stated otherwise**

---

## 3️⃣ Base system setup (Ubuntu)

```bash
apt update && apt upgrade -y
apt install -y curl wget ca-certificates iproute2 iputils-ping \
               socat conntrack ebtables ethtool \
               containerd
```

Why:

* Kubernetes **needs these binaries**
* Android userland does NOT provide them

Checkpoint:

```bash
containerd --version
```

---

## 4️⃣ Disable swap (mandatory)

```bash
swapoff -a
```

Verify:

```bash
free -m
```

If swap > 0 → fix before continuing.

---

## 5️⃣ Download k3s (ARM64)

```bash
mkdir -p /opt/k3s
cd /opt/k3s

curl -Lo k3s https://github.com/k3s-io/k3s/releases/latest/download/k3s-arm64
chmod +x k3s
```

Verify:

```bash
./k3s --version
```

---

## 6️⃣ Start k3s (correct flags)

This avoids Android weirdness and keeps it light.

```bash
./k3s server \
  --data-dir=/opt/k3s/data \
  --disable=traefik \
  --disable=servicelb \
  --disable=local-storage \
  --write-kubeconfig=/opt/k3s/kubeconfig.yaml \
  --write-kubeconfig-mode=644
```

⚠️ Leave this running.

If it exits → read logs, don’t guess.

---

## 7️⃣ Verify cluster (phone)

Open **new shell** inside Ubuntu:

```bash
export KUBECONFIG=/opt/k3s/kubeconfig.yaml
kubectl get nodes
kubectl get pods -A
```

Expected:

* 1 node (Ready)
* Core system pods running

If not → **do not continue**

---

## 8️⃣ Expose API to laptop

Find phone IP:

```bash
ip a show wlan0
```

Edit kubeconfig:

```bash
sed -i 's/127.0.0.1/<PHONE_IP>/' /opt/k3s/kubeconfig.yaml
```

---

## 9️⃣ Copy kubeconfig to laptop

From phone:

```bash
cat /opt/k3s/kubeconfig.yaml
```

On laptop:

```bash
mkdir -p ~/.kube
nano ~/.kube/phone-k3s.yaml
```

Paste content.

Test from laptop:

```bash
kubectl --kubeconfig ~/.kube/phone-k3s.yaml get nodes
```

🎯 **Checkpoint**
If this works:

> You are officially running a **remote Kubernetes cluster**

---

## 10️⃣ What you do NOT do yet

❌ ArgoCD
❌ KIND
❌ Helm
❌ CI
❌ Prometheus

You **practice Kubernetes fundamentals here first**:

* Pods
* Deployments
* Services
* Namespaces
* Failures

---

# STAGE 2 — Laptop KIND + ArgoCD (LEVEL UP)

You only start this **after Stage 1 is boringly stable**.

---

## Goal of Stage 2

* Laptop becomes **GitOps control plane**
* Phone becomes **managed cluster**
* Git becomes the *only* control mechanism

---

## 11️⃣ Create KIND cluster (laptop)

```bash
kind create cluster --name gitops-hub
kubectl cluster-info
```

This cluster is **NOT** for apps.

---

## 12️⃣ Install ArgoCD (hub only)

```bash
kubectl create namespace argocd

kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Expose UI:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Login:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 -d
```

---

## 13️⃣ Register phone cluster in ArgoCD

```bash
argocd login localhost:8080
argocd cluster add \
  --kubeconfig ~/.kube/phone-k3s.yaml \
  <phone-context-name>
```

Verify:

* ArgoCD UI → Settings → Clusters → **Connected**

🎯 This is the **multi-cluster moment**

---

## 14️⃣ GitOps repo structure (do not improvise)

```text
gitops/
├── clusters/
│   ├── hub/
│   └── phone/
├── apps/
│   └── demo/
│       ├── deployment.yaml
│       └── service.yaml
```

This structure scales cleanly.

---

## 15️⃣ Deploy app via GitOps (phone target)

Create ArgoCD Application **on hub cluster**:

```yaml
spec:
  destination:
    server: https://<PHONE_API>
```

ArgoCD applies → phone runs it.

No `kubectl apply`.

---

# 🧠 What This Trains (important)

By doing this, you learn:

* Kubernetes without cloud crutches
* Control plane vs workload separation
* Real GitOps mental models
* Edge cluster management
* Failure recovery by design

This is **platform engineering**, not tutorials.

---

# FINAL RULE (burn this in)

> **If it’s not in Git, it doesn’t exist.**

---

When you’re ready next:

* App-of-Apps
* Environment promotion
* Multi-cluster policies
* Flux comparison
* Disaster recovery drills

Say **“next”** when Stage 1 is done.
