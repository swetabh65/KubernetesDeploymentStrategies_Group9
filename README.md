# 🚀 Kubernetes Rolling Update Strategy – CodeTonic Inc.

## 🏢 Company Background
**CodeTonic Inc.** is a fast-growing SaaS provider offering web analytics dashboards to its clients.  
To maintain high availability and avoid downtime during feature releases, CodeTonic’s DevOps team implements a **Rolling Update deployment strategy** in Kubernetes.

---

## 🎯 **Objective**
To implement a **Rolling Update strategy** where old application versions are gradually replaced by new ones **without downtime**, ensuring continuous service availability during updates.

---

## 🧩 **What is a Rolling Update?**
A **Rolling Update** is a Kubernetes deployment strategy that:
- Replaces Pods **incrementally** with newer versions.  
- Ensures **no full outage** during the upgrade process.  
- Provides **safe and controlled rollouts**.  
- Allows **easy rollback** if something goes wrong.

---

## ⚙️ **How It Works**
Kubernetes controls how many Pods are added or removed during an update using the following configuration:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 1

💡 Explanation

| Parameter           | Meaning                                                    | Example Behavior                    |
| ------------------- | ---------------------------------------------------------- | ----------------------------------- |
| `maxSurge: 1`       | Allows **1 extra pod** (temporary capacity) during updates | Ensures continuous availability     |
| `maxUnavailable: 1` | Allows **1 pod to be down** during the update              | Keeps most pods serving traffic     |
| **Result**          | Rolling replacements occur **gradually**                   | No downtime or service interruption |


🧱 Step-by-Step Implementation
🪄 Step 1: Update Application to v2

Modify your web page to reflect version 2.

index.html

Build and push the updated Docker image:

docker build -t yourdockerhubuser/codetonic:v2 .
docker push yourdockerhubuser/codetonic:v2


🪄 Step 2: Update Kubernetes Deployment YAML

Create or update the deployment manifest file to rolling-update.yaml

Apply the deployment update:

kubectl apply -f rolling-update.yaml


🪄 Step 3: Monitor the Rollout

Watch the update progress:

🪄 Step 3: Monitor the Rollout

Watch the update progress:

kubectl rollout status deployment/codetonic-deployment


Check active pods:
kubectl get pods -l app=codetonic

<img width="915" height="107" alt="image" src="https://github.com/user-attachments/assets/235f1ed8-c8d0-4665-9e25-4c4b4c385f92" />

🪄 Step 4: Test the Live Update

Access the running service:
minikube service codetonic-service

<img width="933" height="247" alt="image" src="https://github.com/user-attachments/assets/f7e94c55-6b45-4b09-86ad-b7b276c9225d" />

http://127.0.0.1:53085/

<img width="1036" height="433" alt="image" src="https://github.com/user-attachments/assets/286e135a-d4e1-4840-a7c2-c330c2c1dae0" />


🪄 Step 5: Rollback (if needed)

If something goes wrong during rollout:
kubectl rollout undo deployment codetonic-deployment

✅ This reverts to the last stable version (v1) seamlessly.


📂 Folder Structure
k8s-deployment-strategies/
├── rolling/
│   ├── index.html        # v2 application version
│   ├── Dockerfile        # Nginx-based image build file
│   ├── rolling-update.yaml
│   └── service.yaml


✅ Testing Checklist

| # | Test Case                  | Expected Result                                  |
| - | -------------------------- | ------------------------------------------------ |
| 1 | Pods update one at a time  | ✅ Old pod terminates only after new one is ready |
| 2 | Service remains accessible | ✅ No downtime during rollout                     |
| 3 | Version verification       | ✅ See both v1 and v2 during transition           |
| 4 | Rollback validation        | ✅ Can revert to v1 using `kubectl rollout undo`  |


🔍 Validation Commands
kubectl get deployments
kubectl get pods -l app=codetonic -w
kubectl rollout history deployment codetonic-deployment

💡 Key Takeaways

Rolling Updates maintain service availability.

You can safely test and monitor live upgrades.

Rollbacks provide instant recovery from failed deployments.

👨‍💻 Author

Swetabh Sonal & Darshit Damania
