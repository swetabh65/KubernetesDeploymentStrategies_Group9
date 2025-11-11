# Canary Deployment on Kubernetes (Minikube) — **codetonic v1 → v2**

This guide walks you through implementing a **Canary deployment** on Kubernetes where a **small subset of users** is routed to a **new version (v2)** while the majority continue to use the **stable version (v1)**. You’ll learn how to deploy, test traffic split, gradually ramp up, and roll back safely.

> **Why Canary?**  
> Canary releases reduce risk by shipping new code to a *small percentage* of traffic first. If metrics and health look good, you increase v2 gradually; if anything looks off, you roll back instantly.

---

## 📌 Prerequisites

- **Minikube** installed and running  
  ```bash
  minikube start
  ```
- **kubectl** configured to talk to your Minikube cluster  
  ```bash
  kubectl cluster-info
  ```
- Docker images for your app:
  - Public images on Docker Hub (recommended for simplicity), **or**
  - Private images + Kubernetes **imagePullSecrets**, **or**
  - Locally built images inside the Minikube Docker daemon (`minikube image build` or `minikube docker-env`).

> **Important:** If using private images, create and attach a pull secret (see the **Image Pull (private repos)** section). If using local images, ensure `imagePullPolicy: IfNotPresent` to prevent re-pulls.

---

## 🧠 Overview of the Canary Strategy

**What it does:**
- Deploys a **new version v2** alongside the current **stable version v1**.
- Routes a **small percentage of traffic** to v2 (the *canary*) for validation.
- After verification, **increase v2 replicas** (or replace v1 entirely).
- **Reduces risk** by catching issues early—before all users are affected.

**How the split works here:**  
We’ll keep a **shared Service** that selects all Pods with `app=codetonic` (both v1 and v2). **Traffic distribution ≈ number of Ready Pods** per version. With 4 replicas of v1 and 1 replica of v2, about **80%** traffic goes to v1 and **20%** to v2 (best-effort).

---

## 🗺️ High-level Architecture

```
+---------------------------+
|        Users/Client       |
+-------------+-------------+
              |
              v
      +-------+--------+
      |  codetonic-service  |  (NodePort Service; selector: app=codetonic)
      +-------+--------+
              |
     +--------+-------------+
     |                      |
     v                      v
+-----------+        +-----------+
| v1 Pods x4|        | v2 Pod x1 |
| version=v1|        |version=v2 |
+-----------+        +-----------+
```

---

## 📂 File Structure

```
k8s-deployment-strategies/
└── canary/
    ├── canary-stable.yaml
    ├── canary-update.yaml
    ├── canary-service.yaml
    └── (optional) Dockerfile / index.html
```

---

## ✅ Step-by-Step Implementation

### Step 1: Deploy the Stable Version (v1)

**`canary-stable.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: codetonic-stable
spec:
  replicas: 4
  selector:
    matchLabels:
      app: codetonic
      version: v1
  template:
    metadata:
      labels:
        app: codetonic
        version: v1
    spec:
      containers:
      - name: codetonic
        image: yourdockerhubuser/codetonic:v1
        ports:
        - containerPort: 80
```

**Apply:**
```bash
kubectl apply -f canary-stable.yaml
```

---

### Step 2: Deploy the Canary Version (v2)

**`canary-update.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: codetonic-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: codetonic
      version: v2
  template:
    metadata:
      labels:
        app: codetonic
        version: v2
    spec:
      containers:
      - name: codetonic
        image: yourdockerhubuser/codetonic:v2
        ports:
        - containerPort: 80
```

**Apply:**
```bash
kubectl apply -f canary-update.yaml
```

---

### Step 3: Expose Both Versions via a Shared Service

**`canary-service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: codetonic-service
spec:
  selector:
    app: codetonic
  ports:
    - port: 80
      targetPort: 80
  type: NodePort
```

This Service forwards traffic to **all Ready Pods** with label `app=codetonic`—which includes both v1 and v2.

**Apply:**
```bash
kubectl apply -f canary-service.yaml
```

---

## 🔎 Step 4: Test Load Distribution

Open the service in a browser:
```bash
minikube service codetonic-service
```

**Refresh several times** — you should mostly see **v1** responses with **occasional v2**.

Or test with `curl`:
```bash
for i in {1..20}; do curl -s $(minikube service codetonic-service --url); echo; done
```

> **Tip:** Actual traffic split is approximate and depends on **how many Ready Pods** exist in each Deployment.


<img width="602" height="251" alt="image" src="https://github.com/user-attachments/assets/f4bf2f51-cef4-4861-9dd0-fac29e86ae5d" />


<img width="602" height="265" alt="image" src="https://github.com/user-attachments/assets/1e1650f7-355a-4c0f-aa41-981ed5a9f94e" />



---

## 📈 Step 5: Gradual Rollout Controls

If v2 is stable and healthy:
```bash
# increase canary traffic (e.g., 40% = 2 v2 Pods vs 3 v1 Pods)
kubectl scale deploy/codetonic-canary --replicas=2

# reduce stable Pods
kubectl scale deploy/codetonic-stable --replicas=3
```

Eventually, you can **promote v2 to stable** by:
- Increasing v2 replicas to match your desired capacity.
- Optionally deleting v1 Deployment once confident.

---

## ⏪ Step 6: Rollback (if needed)

If issues are detected in v2:
```bash
kubectl delete deployment codetonic-canary
```
Now, only **v1** Pods back the Service, and *all* traffic returns to stable safely.

---

## 🔧 Operational Commands & Observability

Check what’s running:
```bash
kubectl get deploy,rs,pods -l app=codetonic -o wide
```

Check Service and endpoints (must not be empty):
```bash
kubectl get svc codetonic-service
kubectl get endpoints codetonic-service
```

Watch rollout and Pod health:
```bash
kubectl rollout status deploy/codetonic-stable
kubectl rollout status deploy/codetonic-canary
kubectl describe pod -l app=codetonic
```

Get a one-off URL for testing:
```bash
minikube service codetonic-service --url
```

---

## 🛡️ Recommended Enhancements (Production-leaning)

While the baseline manifests work, consider adding these for safer rollouts:

### 1) **Readiness Probes** (route traffic only when ready)
```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 2
  periodSeconds: 3
  failureThreshold: 2
```

### 2) **Resource Requests/Limits** (scheduler + stability)
```yaml
resources:
  requests:
    cpu: "50m"
    memory: "64Mi"
  limits:
    cpu: "250m"
    memory: "256Mi"
```

### 3) **Image Pull (private repos)**
Create a pull secret:
```bash
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username="<DOCKERHUB_USERNAME>" \
  --docker-password="<DOCKERHUB_PASSWORD>" \
  --docker-email="<EMAIL>"
```
Reference it in both Deployments:
```yaml
spec:
  template:
    spec:
      imagePullSecrets:
        - name: regcred
```

### 4) **Local Images in Minikube** (no registry)
Use Minikube’s Docker daemon:
```bash
# Linux/macOS
eval $(minikube docker-env)

# Windows PowerShell
minikube -p minikube docker-env --shell powershell | Invoke-Expression

# Build inside Minikube
docker build -t codetonic:v1 ./path/to/v1
docker build -t codetonic:v2 ./path/to/v2
```
Then set images to `codetonic:v1` and `codetonic:v2` with:
```yaml
imagePullPolicy: IfNotPresent
```

### 5) **Diagnostics for Zero Endpoints**
If `kubectl get endpoints codetonic-service` is empty:
- Pods may be **ImagePullBackOff** → fix image name/registry/secret.
- Containers may **not** listen on `:80` → align `containerPort` and `targetPort`.
- Pods may be **not Ready** → add readiness probes or fix app errors.

---

## 🧪 Testing Checklist

- [ ] **v1** receives **most** traffic initially (e.g., 80% with 4 v1 Pods vs 1 v2 Pod).  
- [ ] **v2** is served **occasionally** as the canary.  
- [ ] **Rollout** is easily adjustable by changing **replica counts**.  
- [ ] **Rollback** is trivial by deleting the **canary** Deployment.  
- [ ] Endpoints are **non-empty** and Pods are **Ready**.  
- [ ] Monitoring/logs show **no errors** when increasing v2.

---

## 🧼 Cleanup

```bash
kubectl delete -f canary-service.yaml
kubectl delete -f canary-update.yaml
kubectl delete -f canary-stable.yaml
```

---

## ❓ FAQ

**Q: The Service URL opens, but I get errors or no response.**  
A: Confirm Pods are **Ready** and `kubectl get endpoints codetonic-service` shows IPs. If empty, fix image pulls or probes.

**Q: How do I adjust the exact traffic percentage?**  
A: With this Service approach, it’s **Pod-count based** (approximate). For precise percentages, consider an **Ingress** or **service mesh (Istio/Linkerd)** with weighted routing.

**Q: I’m using private Docker Hub repos—pulls are failing.**  
A: Create an **imagePullSecret** and reference it in the Pod spec (see above).

**Q: Can I run this without Docker Hub?**  
A: Yes—use **Minikube’s Docker daemon** to build local images and set `imagePullPolicy: IfNotPresent`.

---

### 🎯 Summary

You now have a **safe, gradual** deployment workflow: ship v2 to a **small slice**, monitor, **ramp up**, or **roll back** instantly. This pattern keeps users happy and outages rare. Good luck and happy shipping! 🚀
