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
1 — Created the Node.js app

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
**Why:** this is the actual application code. `package.json` records dependencies so Docker and other environments can install them. Express provides a lightweight HTTP server for testing and deployment.

2 — Added `.gitignore`

Contents:
```plaintext
node_modules/
.env
.DS_Store
```
**Why:** prevents committing large or sensitive files. `node_modules/` is rebuilt from `package.json`. `.env` often contains secrets and should not be stored in source control.

3 — Dockerized the app

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

4 — Created Kubernetes manifests

Put these in `k8s/`:

`k8s/deployment.yaml`
```plaintext
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nodejs
  template:
    metadata:
      labels:
        app: nodejs
    spec:
      containers:
      - name: nodejs
        image: <YOUR_IMAGE>:<TAG>
        ports:
        - containerPort: 3000
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
  - host: <YOUR_DOMAIN_OR_HOSTNAME>
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nodejs-service
            port:
              number: 80
```

**Why:** Ingress defines external routing rules (host/path → service). The ingress controller (NGINX) reads these and forwards external HTTP(S) to the matching service in the cluster.

