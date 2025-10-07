# 🚀 Kubernetes Deployment Strategies – CodeTonic Inc.

## 🏢 Company
**CodeTonic Inc.** is a fast-growing SaaS company offering a web analytics dashboard to its clients.  
Recently, users started experiencing **downtime** during feature releases and **broken functionality** after updates.

To resolve this, the DevOps team decided to implement **Kubernetes Deployment Strategies** to ensure:

- ✅ Zero-downtime deployments  
- ✅ Safe rollbacks  
- ✅ Partial rollouts for feature validation  

---

## 🎯 **Objective**
Simulate and test **Blue-Green**, **Canary**, and **Rolling Update** strategies in a Kubernetes environment using a demo application that mimics CodeTonic’s dashboard service.

---

## 🧠 **Pre-requisites**
Ensure the following are installed on your system:
- 🐳 **Docker**
- ☸️ **Kubernetes cluster** (Minikube or Docker Desktop)
- 🔧 **kubectl CLI**
- 🎩 **Helm** (optional for advanced rollouts)
- Basic understanding of **Pods**, **Deployments**, and **Services**

---

## 🏗️ **Approach**

We'll approach the problem in logical phases:

1. **Setup a demo Node.js or Nginx application**
2. **Deploy using multiple strategies:**
   - Rolling Update
   - Blue-Green
   - Canary
3. **Use Kubernetes features** like labels, selectors, and routing to control traffic.
4. **Test** the rollout visually via the app version number and verify with `curl` responses.

---

## ⚙️ **Setup Guide**

### 🧩 Step 1: Setup Kubernetes Cluster

**Option 1 – Using Minikube**
```bash
minikube start
kubectl config use-context minikube

🧩 Step 2: Create Demo Application

We’ll use a simple Nginx-based app showing version numbers.

index.html for version v1 followed by Dockerfile

**Option 2 –Build and Push Image**
```bash
docker build -t yourdockerhubuser/codetonic:v1 .
docker push yourdockerhubuser/codetonic:v1

🧩 Step 3: Create Kubernetes Manifests

  Create a deployment and service YAML file


🧩 Step 4: Apply and Verify

```bash
kubectl apply -f codetonic-deployment.yaml
kubectl apply -f codetonic-service.yaml

**Check the running service:**
```bash
minikube service codetonic-service

http://127.0.0.1:51199/

Welcome to CodeTonic Dashboard v1

<img width="1100" height="583" alt="image" src="https://github.com/user-attachments/assets/2b72fdad-88b2-499f-bf4b-b55a2af7ebb6" />


📂 Project Folder Structure

k8s-deployment-strategies/
│
├── blue-green/
│   ├── v1/
│   ├── v2/
│
├── canary/
│   ├── base-deployment.yaml
│   ├── v2-deployment.yaml
│
├── rolling/
│   ├── rolling-deployment.yaml
│
├── Dockerfile
├── index.html
├── service.yaml
└── README.md


🧪 Testing Strategies
Strategy	Description	Test Method
Rolling Update	Gradually replaces old pods with new ones	kubectl rollout status deployment
Blue-Green	Two versions (blue/green) with one live	Switch service selector to v2
Canary	Small % of traffic goes to new version	Label pods differently, split service traffic


🧾 Expected Output

Zero downtime during updates

Ability to roll back easily

Validate new features on a subset of traffic

👨‍💻 Author

Swetabh Sonal & Darshit Damania



