# Blue-Green Deployment Strategy for Kubernetes

## 📌 Overview
Blue-Green Deployment is a release management strategy that reduces downtime and risk by running two identical production environments, known as **Blue (current version)** and **Green (new version)**.

### ✅ What This Strategy Achieves:
- Zero downtime deployments
- Instant rollback capability
- Safe testing of new version before switching live traffic

### 🔵 Blue Environment (v1)
- Currently serving live traffic

### 🟢 Green Environment (v2)
- New version deployed in parallel
- Tested before switching traffic

---
## 📂 Project File Structure
```
k8s-deployment-strategies/
└── blue-green/
    ├── blue-deployment.yaml
    ├── green-deployment.yaml
    ├── blue-green-service.yaml
    └── Dockerfile / index.html (for both versions)
```

---
## 🚀 Step-by-Step Implementation

### ✨ Step 1: Deploy Blue Version (v1)
**File: `blue-deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: codetonic-blue
spec:
  replicas: 3
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
**Apply the deployment:**
```bash
kubectl apply -f blue-deployment.yaml
```

---
### 🌐 Step 2: Expose Blue Deployment via Service
**File: `blue-green-service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: codetonic-service
spec:
  selector:
    app: codetonic
    version: v1
  ports:
    - port: 80
      targetPort: 80
  type: NodePort
```
**Apply the service:**
```bash
kubectl apply -f blue-green-service.yaml
```
**To access service (Minikube users):**
```bash
minikube service codetonic-service
```

---
### 🟢 Step 3: Deploy Green Version (v2)
**File: `green-deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: codetonic-green
spec:
  replicas: 3
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
**Apply the deployment:**
```bash
kubectl apply -f green-deployment.yaml
```

---
### 🔍 Step 4: Smoke Test Green Deployment
```bash
kubectl get pods -l version=v2
kubectl port-forward deployment/codetonic-green 8080:80
```
Visit:
```
http://localhost:8080
```
You should see output indicating **Version 2 (Green)** is running.

---
### 🔄 Step 5: Switch Traffic to Green
Edit the selector in `blue-green-service.yaml` to point to `version: v2`:
```yaml
spec:
  selector:
    app: codetonic
    version: v2
```
**Apply the update:**
```bash
kubectl apply -f blue-green-service.yaml
```
✅ All production traffic now goes to the Green environment (v2).

---
### 🔁 Step 6: Rollback to Blue (if needed)
Simply revert the selector back to `version: v1` and reapply:
```yaml
spec:
  selector:
    app: codetonic
    version: v1
```
Apply rollback:
```bash
kubectl apply -f blue-green-service.yaml
```

🚀 Instant rollback with zero downtime.

---
## ✅ Testing Checklist
| Test Case | Expected Result |
|----------|----------------|
| Blue (v1) responds via service initially | ✅ Yes |
| Green (v2) deployed but isolated | ✅ Yes |
| Green passes smoke test | ✅ Yes |
| Switch Service selector to v2 | ✅ Traffic goes to v2 |
| Rollback by switching back to v1 | ✅ Instant recovery |

---
## 📌 Key Benefits of Blue-Green Deployment
- **Zero Downtime:** Users never experience interruption.
- **Instant Rollback:** Quickly revert to previous stable version.
- **Safe Testing:** Green version is fully tested before going live.
- **Reduced Risk:** New version doesn’t impact users until explicitly promoted.

---
## 🧠 Conclusion
Blue-Green Deployment is a powerful strategy for production environments where uptime and reliability are critical. By simply switching service selectors, we can manage deployments with **confidence, flexibility, and zero downtime**.

> 💡 Next Step: Automate the switch using CI/CD pipelines for full production readiness!
