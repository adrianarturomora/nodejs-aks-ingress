# Node.js AKS Ingress Project

This project demonstrates deploying a Node.js application to **Azure Kubernetes Service (AKS)** with **NGINX Ingress** for routing HTTP traffic. It includes all the necessary configurations for containerization, deployment, and Kubernetes ingress setup.

## Project Overview

- **Node.js Application** – A simple web application that serves HTTP requests.
- **Dockerized** – The application is containerized using Docker for easy deployment.
- **Kubernetes Deployment** – Manifests for deploying the app to AKS.
- **NGINX Ingress** – Configured to expose the application externally with routing and host/path rules.
- **Environment Variables** – Stored in a `.env` file (ignored by Git for security).
- **Git Version Control** – Repository initialized with `.gitignore` configured for `node_modules/`, `.env`, and `.DS_Store`.

## Project Structure
```plaintext
nodejs-aks-ingress/
│
├─ src/                 # Node.js application source code
├─ Dockerfile           # Docker configuration for the container image
├─ .gitignore           # Files to ignore (node_modules, .env, etc.)
├─ .env                 # Local environment variables (not committed)
├─ k8s/                 # Kubernetes manifests
│   ├─ deployment.yaml
│   ├─ service.yaml
│   └─ ingress.yaml
└─ README.md
```
## Project Steps
### 1 — Created the Node.js app

Commands / files:
```plaintext
mkdir src
cd src
npm init -y
npm install express
```
Created `src/index.js`:
```plaintext
const express = require('express');
const app = express();
const port = process.env.PORT || 3000;
app.get('/', (req, res) => res.send('Hello, AKS!'));
app.listen(port, () => console.log(`Server running on ${port}`));
```
**Why:** This is the actual application code. `package.json` records dependencies so Docker and other environments can install them. Express provides a lightweight HTTP server for testing and deployment.

### 2 — Added `.gitignore`

Contents:
```plaintext
node_modules/
.env
.DS_Store
```
**Why:** Prevents committing large or sensitive files. `node_modules/` is rebuilt from `package.json`. `.env` often contains secrets and should not be stored in source control.

### 3 — Dockerized the app

Dockerfile (root):
```plaintext
FROM node:18-alpine
WORKDIR /app
COPY src/package*.json ./
RUN npm install
COPY src/ ./
EXPOSE 3000
CMD ["node", "index.js"]
```

Commands:
```plaintext
docker build -t nodejs-aks-ingress .
docker run -p 3000:3000 nodejs-aks-ingress   # test locally
```

**Why:** Docker creates a portable image that contains your app and its runtime. This ensures the same app behavior whether on your laptop, a CI environment, or in Kubernetes. `COPY package*.json` + `npm install` optimizes layer caching so rebuilding is faster.

### 4 — Created Kubernetes manifests

Put these in `k8s/`:

`k8s/deployment.yaml`
```plaintext
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nodejs-app
  template:
    metadata:
      labels:
        app: nodejs-app
    spec:
      containers:
      - name: nodejs-app
        image: myacradrian9876.azurecr.io/nodejs-aks-demo:v1
        ports:
        - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: nodejs-app
spec:
  selector:
    app: nodejs-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: ClusterIP
```

**Why:** Deployment describes desired state: how many pods, which image, and restart/update behavior. Kubernetes will keep the specified number of replicas running and handle restarts/rolling updates.

`k8s/ingress.yaml`
```plaintext
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nodejs-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nodejs-app
            port:
              number: 80
```

**Why:** Ingress defines external routing rules (host/path → service). The ingress controller (NGINX) reads these and forwards external HTTP(S) to the matching service in the cluster.

### 5 — Provisioned AKS (created cluster)

Commands:
```plaintext
az group create --name MyResourceGroup --location eastus
az aks create --resource-group MyResourceGroup --name myAKSCluster --node-count 2 --enable-managed-identity --generate-ssh-keys
az aks get-credentials --resource-group MyResourceGroup --name myAKSCluster
```
**Why:** AKS is a managed Kubernetes service on Azure. az aks get-credentials configures your kubectl context so subsequent kubectl commands target the new cluster.

### 6 — Installed NGINX Ingress controller (on the cluster)

Command:
```plaintext
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.0/deploy/static/provider/cloud/deploy.yaml
```
**Why:** Kubernetes’ `Ingress` resource is only rules; the controller (NGINX here) implements those rules and performs the actual routing of external traffic into the cluster. Without a controller, Ingress resources do nothing.

### 7 — Prepared image registry & pushed image

**Options:** Use Azure Container Registry (ACR) or Docker Hub. Example (ACR):

1. Created ACR:
```plaintext
az acr create --resource-group MyResourceGroup --name myacradrian9876 --sku Basic
```
2. Tagged & pushed:
```plaintext
docker tag nodejs-aks-ingress myacradrian9876.azurecr.io/nodejs-aks-ingress:v1
docker push myacradrian9876.azurecr.io/nodejs-aks-ingress:v1
```
3. Granted AKS access to ACR:
```plaintext
az aks update -n myAKSCluster -g MyResourceGroup --attach-acr myacradrian9876
```
**Why:** Kubernetes pulls images from a registry. ACR is Azure’s registry — pushing there centralizes your images and allows AKS to pull them. `--attach-acr` gives AKS permission to pull private images.

### 8 — Deployed manifests to AKS

Commands:
```plaintext
# ensure the image in deployment.yaml matches the pushed image: myacradrian9876.azurecr.io/...
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/ingress.yaml
```
**Why:** `kubectl apply` tells Kubernetes to create or update the resources to match the YAML definitions. This spins up pods, creates the service, and configures ingress rules.

### 9 — Verified and debugged

Commands:
```plaintext
kubectl get pods -o wide
kubectl get svc
kubectl get ingress
kubectl describe pod <pod-name>
kubectl logs <pod-name>        # check container logs
```
**Why:** These commands check the running state and help debug issues like image pull errors, containers crashing, or misconfigured services.

### 10 — Accessed the app externally

Checked the external IP of the ingress controller service:
```plaintext
kubectl get svc -n ingress-nginx
```
- Mapped the host in ingress.yaml to the external IP via DNS or /etc/hosts entry for testing.
- Opened the hostname in a browser.
**Why:** The ingress controller gets a cloud load-balancer IP. The Ingress resource routes traffic from that external IP → ingress controller → service → pods.

### 11 — Environment variables / secrets
- Used a .env locally.

For Kubernetes, I used Secrets or ConfigMaps:
```plaintext
kubectl create secret generic my-secret --from-literal=API_KEY=xxxx
```
**Why:** Secrets and config maps keep sensitive info and configuration separate from code, and allow different values in different environments (dev/stage/prod).

### 12 — Cleanup (delete Azure resources when finished)

Commands:
```plaintext
az group delete --name MyResourceGroup --yes --no-wait
```
**Why:** Deleting the resource group removes AKS, ACR, and any other resources inside it to avoid ongoing charges.

## Project Summary

In this project, I built a simple Node.js application, containerized it with Docker, and deployed it onto an Azure Kubernetes Service (AKS) cluster. I configured Kubernetes resources—including a Deployment, Service, and an NGINX Ingress—to run the application and expose it externally. I also set up Azure Container Registry (ACR) to store my Docker image and granted AKS access to pull from it. After deploying everything to the cluster, I verified that the pods, services, and ingress were working correctly, mapped the external IP, and accessed the app through the configured host. Finally, I used Kubernetes Secrets for environment variables and cleaned up the Azure resources when finished.
