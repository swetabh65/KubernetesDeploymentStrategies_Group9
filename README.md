# Testing & Monitoring Playbook — Rolling, Blue‑Green, and Canary (Minikube)

This guide helps you **analyze, test, and monitor** each Kubernetes deployment strategy under realistic traffic. You’ll verify routing, observe pod behavior, check logs/events, and practice failure handling with fast rollbacks.

> Target app label: `app=codetonic`  
> Default Service: `codetonic-service` (NodePort, selector `app=codetonic`)  
> Cluster: Minikube (local)  

---

## 🔧 Useful One‑Liners (keep these handy)

```bash
# Watch pods live (Linux/macOS)
kubectl get pods -l app=codetonic -w

# PowerShell alternative (Windows)
while ($true) { kubectl get pods -l app=codetonic; Start-Sleep -Seconds 2; Clear-Host }

# Get deployments, replicasets, pods (wide)
kubectl get deploy,rs,pods -l app=codetonic -o wide

# Service + endpoints (must not be empty)
kubectl get svc codetonic-service
kubectl get endpoints codetonic-service

# Tail logs for all pods
kubectl logs -l app=codetonic -f --max-log-requests=20

# Describe the most recent failing pod (replace name as needed)
kubectl describe pod <pod-name>
```

---

## 🚚 Rolling Update — Testing Procedure

**Goal:** Ensure pods are **gradually updated** with **zero downtime**, and at some point you observe **mixed versions** during rollout.

### Steps

1) **Start/observe rollout**
```bash
kubectl rollout status deployment/codetonic-deployment
```

2) **Watch pods changing versions**
```bash
kubectl get pods -l app=codetonic -w
```

3) **Hit the service repeatedly**
- **Linux/macOS (bash/zsh)**
  ```bash
  for i in {1..20}; do curl -s $(minikube service codetonic-service --url); done
  ```
- **Windows PowerShell**
  ```powershell
  1..20 | ForEach-Object { Invoke-WebRequest -UseBasicParsing $(minikube service codetonic-service --url) | Select-Object -ExpandProperty Content }
  ```

### Validate
- ✅ **No downtime** (every request returns 200/expected body)
- ✅ **Mixed responses** (old & new versions) **during** rollout
- ✅ **All pods converge** to the new version at the end

### Troubleshooting
- If requests fail: `kubectl get endpoints codetonic-service` (empty? → pods not Ready)
- Check events/logs: `kubectl describe pod <name>` and `kubectl logs <name>`

---

## 🟦🟩 Blue‑Green — Testing Procedure

**Goal:** Ensure traffic goes **only to the intended environment**; switching (cutover) is **instant** and **reversible**.

> Typical labels:  
> Blue (live): `app=codetonic, env=blue`  
> Green (staging): `app=codetonic, env=green`

### Steps

1) **Access live app via Service**
```bash
minikube service codetonic-service
```

2) **Smoke test Green (not receiving prod traffic yet)**  
Port‑forward to Green deployment:
```bash
kubectl port-forward deployment/codetonic-green 8080:80
# Now open http://localhost:8080 to validate the new version
```

3) **Switch Service selector** to Green for cutover
```bash
# Option A — patch selector (instant)
kubectl patch svc codetonic-service -p '{"spec":{"selector":{"app":"codetonic","env":"green"}}}'

# Option B — edit YAML and apply
# spec.selector:
#   app: codetonic
#   env: green
```

4) **Test again via Service URL**
```bash
minikube service codetonic-service --url
```

### Validate
- ✅ **Instant cutover** (no downtime)
- ✅ **Only Green** receives traffic post‑switch
- ✅ **Easy rollback** by pointing selector back to Blue

### Rollback
```bash
kubectl patch svc codetonic-service -p '{"spec":{"selector":{"app":"codetonic","env":"blue"}}}'
```

---

## 🐤 Canary — Testing Procedure

**Goal:** Route a **small percentage** of traffic to **v2** for observation, while most users remain on **v1**.

> Example: v1 replicas = 4, v2 replicas = 1 → roughly **80%** v1 and **20%** v2

### Steps

1) **Check pod distribution**
```bash
kubectl get pods -l app=codetonic --show-labels
kubectl get endpoints codetonic-service
```

2) **Generate requests and count responses**  
- **Linux/macOS**
  ```bash
  URL=$(minikube service codetonic-service --url)
  for i in {1..200}; do curl -s "$URL"; done | sort | uniq -c
  ```
- **Windows PowerShell**
  ```powershell
  $u = minikube service codetonic-service --url
  $resp = 1..200 | ForEach-Object { (Invoke-WebRequest -UseBasicParsing $u).Content }
  $resp | Group-Object | Select-Object Count, Name | Sort-Object Name
  ```

### Validate
- ✅ **80–90% v1** responses, **10–20% v2** (given 4:1 replicas)
- ✅ **Stable** responses under modest load
- ✅ Any **v2 issues** show up early without impacting most users

### Adjust traffic
```bash
# Increase canary share
kubectl scale deploy codetonic-canary --replicas=2

# Decrease stable
kubectl scale deploy codetonic-stable --replicas=3
```

### Rollback canary
```bash
kubectl delete deploy codetonic-canary
```

---

## 💥 Simulating Failures (Realistic Drills)

### A) Faulty v2 (crash / bad port / failing readiness)

**Option 1 — Crash immediately**
```yaml
# In codetonic v2 container spec:
command: ["/bin/sh","-c"]
args: ["echo 'starting v2 and crashing'; exit 1"]
```

**Option 2 — Wrong listening port** (Service expects 80)
```yaml
ports:
  - containerPort: 8080   # mismatch on purpose
```

**Option 3 — Failing readiness probe**
```yaml
readinessProbe:
  httpGet: { path: /healthz, port: 80 }
  initialDelaySeconds: 1
  periodSeconds: 2
  failureThreshold: 1
```

**Apply and observe:**
```bash
kubectl apply -f canary-update.yaml
kubectl get pods
kubectl describe pod <v2-pod>
kubectl logs <v2-pod>
kubectl get endpoints codetonic-service
```

**What to validate**
- ⏱️ **Detection speed:** Pod becomes `CrashLoopBackOff` or stays `NotReady` quickly
- 🧯 **Blast radius:** Most users are unaffected (v1 still serving)
- 🔁 **Rollback:** `kubectl delete deploy codetonic-canary` restores full stability

---

## 📊 Monitoring & Metrics (Optional Enhancements)

> For deeper, production‑like insights you can integrate the following stacks. These are optional for the assignment but great for learning.

### Prometheus + Grafana
- **Prometheus** scrapes Kubernetes metrics (CPU, memory, pod restarts, readiness).
- **Grafana** visualizes dashboards (Pod availability, request rate, error rate).

Quickstart (Helm):
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install kps prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace
# Access Grafana
kubectl port-forward svc/kps-grafana -n monitoring 3000:80
# open http://localhost:3000
```

### Istio + Kiali (for weighted routing & traffic graphs)
- **Istio** enables **precise traffic splitting** (e.g., 10% v2) and retries/timeouts.
- **Kiali** provides a service graph to visualize traffic flow and errors.

### Sentry / Datadog / New Relic
- Add application‑level instrumentation for **exceptions**, **APM**, and **alerts**.

> Note: If you stick with the baseline Service approach, traffic splits are **replica‑based** (approx). For **exact percentages**, use **Ingress/Istio** with weighted routing.

---

## 🧪 Acceptance Checklists

### Rolling
- [ ] `kubectl rollout status` shows **progressing** → **completed**
- [ ] Requests continue to succeed during rollout (**no downtime**)
- [ ] Mixed versions visible mid‑rollout; all new version at end

### Blue‑Green
- [ ] Green validated via **port‑forward** before go‑live
- [ ] Service selector switched; **instant cutover**
- [ ] Rollback by swapping selector back to Blue

### Canary
- [ ] Initial split ≈ **replica ratio** (e.g., 80/20)
- [ ] v2 behavior/metrics monitored
- [ ] Easy rollback by **deleting canary**

---

## 📋 Summary Table — Strategy Comparison

| Strategy   | Downtime | Rollback Complexity      | Test Isolation | Risk Level | Traffic Control |
|------------|----------|--------------------------|----------------|------------|-----------------|
| Rolling    | None     | Medium                   | No             | Medium     | Low             |
| Blue-Green | None     | Easy (via selector)      | Yes            | Low        | Full swap       |
| Canary     | None     | Easy (delete canary)     | Partial        | Very Low   | Granular        |

---

## 🧼 Cleanup

```bash
# Canary
kubectl delete deployment codetonic-canary --ignore-not-found=true

# Blue/Green (if created)
kubectl delete deployment codetonic-blue --ignore-not-found=true
kubectl delete deployment codetonic-green --ignore-not-found=true

# Rolling (single deployment)
kubectl delete deployment codetonic-deployment --ignore-not-found=true

# Service
kubectl delete svc codetonic-service --ignore-not-found=true
```

---

### ✅ Tips for Reliable Tests

- Add **readinessProbes** so the Service only routes to healthy Pods.
- Keep **logs open** during rollouts: `kubectl logs -l app=codetonic -f`.
- Validate **endpoints** after each change.
- Prefer **small, frequent** changes; practice rollback until it’s muscle memory.
