# gitops-argocd-RE

Architecture 
Your Laptop
   |
   | SSH
   v
AWS EC2 (Ubuntu)
   |
   | Minikube
   v
Kubernetes Cluster
   |
   | Argo CD watches
   v
GitHub Repo (YAML files)

2️⃣ Prerequisites
AWS EC2 Requirements

Ubuntu 20.04 or 22.04

Instance type: t2.medium (important)

Storage: at least 20 GB

Security Group:

SSH (22) – Your IP

HTTP (80) – Optional

Custom TCP 8080 – Optional (Argo CD UI)

3️⃣ Login to EC2
ssh ubuntu@<EC2_PUBLIC_IP>

4️⃣ Install Basic Tools (One Time)
Update system
sudo apt update -y

Install Docker
sudo apt install docker.io -y
sudo usermod -aG docker ubuntu
newgrp docker


Check:

docker --version

Install kubectl
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/


Check:

kubectl version --client

Install Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube-linux-amd64
sudo mv minikube-linux-amd64 /usr/local/bin/minikube


Check:

minikube version

Install Git
sudo apt install git -y

5️⃣ Start Kubernetes (Minikube)
minikube start --driver=docker


Verify:

kubectl get nodes


 Node should be Ready

6️⃣ Create Sample Application (No Coding)
Create project directory
mkdir gitops-demo
cd gitops-demo

Create Kubernetes manifest folder
mkdir manifests
cd manifests

Create Deployment YAML
nano deployment.yaml


Paste:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80


Save & exit.

Create Service YAML
nano service.yaml


Paste:

apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80

7️⃣ Push Manifests to GitHub (GitOps Source)
On GitHub

Create repo: gitops-argocd-demo

Public repository

Push code from EC2
cd ..
git init
git add .
git commit -m "Initial Kubernetes manifests"
git branch -M main
git remote add origin https://github.com/<YOUR_USERNAME>/gitops-argocd-demo.git
git push -u origin main

8️⃣ Install Argo CD in Kubernetes
Create namespace
kubectl create namespace argocd

Install Argo CD
kubectl apply -n argocd \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml


Wait:

kubectl get pods -n argocd


All pods should be Running

9️⃣ Access Argo CD UI (EC2 Friendly)
Port forward
kubectl port-forward svc/argocd-server -n argocd 8080:443


Open browser:

https://<EC2_PUBLIC_IP>:8080


 Allow inbound port 8080 in Security Group

Get Admin Password
kubectl get secret argocd-initial-admin-secret -n argocd \
-o jsonpath="{.data.password}" | base64 --decode


Login:

Username: admin

Password: (above output)

🔟 Create Argo CD Application (GitOps Link)
In Argo CD UI → NEW APP
General

Application Name: nginx-gitops

Project: default

Sync Policy: Automatic

Source

Repo URL: https://github.com/<YOUR_USERNAME>/gitops-argocd-demo

Revision: main

Path: manifests

Destination

Cluster: https://kubernetes.default.svc

Namespace: default

Click CREATE

App deploys automatically

1️⃣1️⃣ Verify Deployment
kubectl get pods
kubectl get svc


Access app:

minikube service nginx-service

1️⃣2️⃣ GitOps Demo – Auto Deploy Change
Modify Image Version
nano manifests/deployment.yaml


Change:

image: nginx:1.26

Push change
git add .
git commit -m "Upgrade nginx image"
git push


Open Argo CD UI

App auto syncs

Pod restarts automatically

No kubectl apply used ❌
Only Git change ✅
