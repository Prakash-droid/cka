# 🧩 Problem Statement (NetworkPolicy Configuration)

<details>
<summary>Click to expand the problem statement</summary>

A Kubernetes cluster has frontend and backend Deployments running in separate namespaces. The backend namespace has a default-deny NetworkPolicy applied. Your task is to analyze the deployments and apply the correct NetworkPolicy from the provided samples to enable communication between them.

Your task is to:

1. Connect to the **correct host**
2. Analyze the frontend and backend Deployments to determine NetworkPolicy requirements
3. Review the NetworkPolicy YAML samples in the **`~/netpol`** directory
4. Apply the NetworkPolicy that enables communication **without being overly permissive**

```
[candidate@base]$ ssh <hostname>
```

### 📋 Configure the following:

1. Allow communication between the **`frontend`** Deployment (in the **`frontend`** namespace) and the **`backend`** Deployment (in the **`backend`** namespace)
2. Ensure the NetworkPolicy is **not overly permissive**
3. Review the NetworkPolicy YAML samples in **`~/netpol`** directory
   * Do **not** delete or modify the samples — only apply one of them
   * Modifying samples may result in a lower score
4. Do **not** delete or modify existing default NetworkPolicies (e.g., those denying all inbound/outbound traffic)
   * Doing so may result in a score of 0

</details>

---

# 🛠 Setup Practice Scenario

<details>
<summary>Click to expand setup steps</summary>

| Script | Purpose |
|--------|---------|
| `setup_q15_netpol.sh` | Creates namespaces, deployments, default-deny policy and netpol samples — run before practice (re-run anytime) |

---

### 📥 Script : setup_q15_netpol.sh

```bash
#!/bin/bash
# ============================================================
# CKA Practice | Q15: NetworkPolicy Configuration
# setup_q15_netpol.sh
# Re-run safe: only resets candidate work on subsequent runs
# Run on: controlplane node
# ============================================================

set -euo pipefail

NS_FRONTEND="frontend"
NS_BACKEND="backend"
NETPOL_DIR="$HOME/netpol"

echo ""
echo "==========================================="
echo " CKA Q15 — Setting up NetworkPolicy scenario..."
echo "==========================================="
echo ""

# ── Step 1: Reset only candidate work ────────────────────────
echo "[1/5] Resetting candidate work from previous attempt..."

# Delete only candidate-applied NetworkPolicies (not default-deny-all)
kubectl delete networkpolicy netpol-1 -n "$NS_BACKEND" --ignore-not-found
kubectl delete networkpolicy netpol-2 -n "$NS_BACKEND" --ignore-not-found
kubectl delete networkpolicy netpol-3 -n "$NS_BACKEND" --ignore-not-found

# Always recreate netpol dir so samples are fresh
rm -rf "$NETPOL_DIR"

echo "  Candidate work cleared."

# ── Step 2: Create namespaces if not present ──────────────────
echo "[2/5] Checking namespaces..."
if kubectl get namespace "$NS_FRONTEND" &>/dev/null && \
   kubectl get namespace "$NS_BACKEND" &>/dev/null; then
    echo "  Namespaces already exist — skipping."
else
    echo "  Creating namespaces..."
    kubectl get namespace "$NS_FRONTEND" &>/dev/null || \
        kubectl create namespace "$NS_FRONTEND"
    kubectl get namespace "$NS_BACKEND" &>/dev/null || \
        kubectl create namespace "$NS_BACKEND"
    echo "  Namespaces created."
fi

# ── Step 3: Create deployments and services if not present ────
echo "[3/5] Checking deployments..."
if kubectl get deployment frontend -n "$NS_FRONTEND" &>/dev/null && \
   kubectl get deployment backend -n "$NS_BACKEND" &>/dev/null; then
    echo "  Deployments already exist — skipping."
else
    echo "  Creating deployments and services..."
    kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: $NS_FRONTEND
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: nginx:1.27
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  namespace: $NS_FRONTEND
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: $NS_BACKEND
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: nginx:1.27
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: $NS_BACKEND
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80
EOF

    echo "  Waiting for deployments to be Ready..."
    kubectl wait --for=condition=available deployment/frontend \
        -n "$NS_FRONTEND" --timeout=120s
    kubectl wait --for=condition=available deployment/backend \
        -n "$NS_BACKEND" --timeout=120s
    echo "  Deployments are Ready."
fi

# ── Step 4: Apply default-deny-all if not present ─────────────
echo "[4/5] Checking default-deny-all NetworkPolicy..."
if kubectl get networkpolicy default-deny-all -n "$NS_BACKEND" &>/dev/null; then
    echo "  default-deny-all already exists — skipping."
else
    echo "  Applying default-deny-all NetworkPolicy..."
    kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: $NS_BACKEND
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF
    echo "  default-deny-all NetworkPolicy applied."
fi

# ── Step 5: Create ~/netpol sample files ──────────────────────
echo "[5/5] Creating NetworkPolicy sample files in '$NETPOL_DIR'..."

mkdir -p "$NETPOL_DIR"

# netpol1.yaml — overly permissive (targets ALL pods in backend namespace)
cat > "$NETPOL_DIR/netpol1.yaml" <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: netpol-1
  namespace: $NS_BACKEND
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: $NS_FRONTEND
EOF

# netpol2.yaml — correct answer (precise selectors, least permissive)
cat > "$NETPOL_DIR/netpol2.yaml" <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: netpol-2
  namespace: $NS_BACKEND
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: $NS_FRONTEND
      podSelector:
        matchLabels:
          app: frontend
EOF

# netpol3.yaml — wrong target + overly permissive IP range
cat > "$NETPOL_DIR/netpol3.yaml" <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: netpol-3
  namespace: $NS_BACKEND
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: $NS_FRONTEND
      podSelector:
        matchLabels:
          app: frontend
    - ipBlock:
        cidr: 10.0.0.0/24
EOF

echo "  Sample files created:"
ls -1 "$NETPOL_DIR"

echo ""
kubectl get ns "$NS_FRONTEND" "$NS_BACKEND" --show-labels
echo ""
kubectl get networkpolicies -n "$NS_BACKEND"
echo ""
echo "✅ Practice scenario is ready!"
echo ""
echo "   Namespaces  : $NS_FRONTEND | $NS_BACKEND"
echo "   Netpol dir  : ~/netpol (netpol1.yaml, netpol2.yaml, netpol3.yaml)"
echo "   Default deny: applied in $NS_BACKEND"
echo "   💪 Analyze deployments and apply the correct NetworkPolicy!"
echo ""
```

</details>

---

# ✅ Solution

<details>
<summary>Click to expand the solution</summary>

### Step 1: Test connection BEFORE applying NetworkPolicy

```bash
# Get frontend pod name
FRONTEND_POD=$(kubectl get pod -n frontend -l app=frontend \
    -o jsonpath='{.items[0].metadata.name}')

# Test connection from frontend pod to backend service
kubectl exec "$FRONTEND_POD" -n frontend -- \
    curl  --max-time 3 http://backend-svc.backend.svc.cluster.local
```

Expected output — connection **fails** due to default-deny-all policy:

```
curl: (28) Connection timed out after 3001 milliseconds
```

This confirms the default-deny-all NetworkPolicy is blocking traffic. ✅

---

### Step 2: Analyze namespace labels

```bash
kubectl get ns frontend backend --show-labels
```

Expected output:

```
NAME       STATUS   AGE   LABELS
frontend   Active   ...   kubernetes.io/metadata.name=frontend
backend    Active   ...   kubernetes.io/metadata.name=backend
```

Key label: `kubernetes.io/metadata.name=<namespace-name>` — built-in label on all namespaces.

---

### Step 3: Analyze pod labels

```bash
kubectl get pod -n frontend --show-labels
```

```bash
kubectl get pod -n backend --show-labels
```

Expected output:

```
# frontend namespace
NAME               LABELS
frontend-xxx       app=frontend,...

# backend namespace
NAME               LABELS
backend-xxx        app=backend,...
```

Key labels: `app=frontend` and `app=backend`.

---

### Step 4: Check existing NetworkPolicies

```bash
kubectl get networkpolicies -n backend
```

Expected output:

```
NAME               POD-SELECTOR   AGE
default-deny-all   <none>         ...
```

`default-deny-all` denies all inbound traffic to pods in `backend`. Do **not** delete or modify this policy.

---

### Step 5: Review the NetworkPolicy samples

```bash
cat ~/netpol/netpol1.yaml
```

❌ **Issue:** `podSelector: {}` targets **all pods** in `backend` namespace — overly permissive.

---

```bash
cat ~/netpol/netpol2.yaml
```

✅ **Correct:**
- `podSelector.matchLabels.app: backend` — targets only `backend` pods precisely
- Combines `namespaceSelector` + `podSelector` — allows traffic only from `frontend` pods in `frontend` namespace
- Least permissive, meets requirements exactly

---

```bash
cat ~/netpol/netpol3.yaml
```

❌ **Issues:**
- Targets `app: database` pods — wrong target, not the backend deployment
- Allows additional `10.0.0.0/24` IP range — overly permissive

---

### Step 6: Apply the correct NetworkPolicy

```bash
kubectl apply -f ~/netpol/netpol2.yaml
```

Expected output:

```
networkpolicy.networking.k8s.io/netpol-2 created
```

---

### Step 7: Verify the NetworkPolicy is applied

```bash
kubectl get networkpolicies -n backend
```

Expected output:

```
NAME               POD-SELECTOR   AGE
default-deny-all   <none>         ...
netpol-2           app=backend    10s
```

---

### Step 8: Test connection AFTER applying NetworkPolicy

```bash
FRONTEND_POD=$(kubectl get pod -n frontend -l app=frontend \
    -o jsonpath='{.items[0].metadata.name}')

kubectl exec "$FRONTEND_POD" -n frontend -- \
    curl --max-time 3 http://backend-svc.backend.svc.cluster.local
```

Expected output — connection **succeeds**:

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

Communication between `frontend` and `backend` is now working. ✅

</details>

---

# ↩️ Restore Cluster State

<details>
<summary>Click to expand reset steps</summary>

| Script | Purpose |
|--------|---------|
| `reset_q15_netpol.sh` | Removes namespaces and netpol directory — run before moving to next problem |

---

### 🧹 Script : reset_q15_netpol.sh

```bash
#!/bin/bash
# ============================================================
# CKA Practice | Q15: NetworkPolicy Configuration
# reset_q15_netpol.sh
# Removes namespaces and netpol sample directory
# Run on: controlplane node
# ============================================================

set -euo pipefail

NS_FRONTEND="frontend"
NS_BACKEND="backend"
NETPOL_DIR="$HOME/netpol"

echo ""
echo "==========================================="
echo " CKA Q15 — Restoring cluster to baseline..."
echo "==========================================="
echo ""

# ── Delete namespaces ─────────────────────────────────────────
echo "[1/2] Deleting namespaces '$NS_FRONTEND' and '$NS_BACKEND'..."
kubectl delete namespace "$NS_FRONTEND" --ignore-not-found
kubectl delete namespace "$NS_BACKEND" --ignore-not-found
echo "  Namespaces and all resources removed."

# ── Remove netpol directory ───────────────────────────────────
echo "[2/2] Removing netpol directory '$NETPOL_DIR'..."
rm -rf "$NETPOL_DIR"
echo "  Netpol directory removed."

echo ""
echo "✅ Cluster is back to baseline!"
echo "   Safe to move to next problem. 🚀"
echo ""
```

</details>

---
