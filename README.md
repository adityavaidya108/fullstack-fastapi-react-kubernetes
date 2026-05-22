# 🚀 Kubernetes Deployment on AWS EC2 with Rancher, Docker & Jenkins

This guide walks you through **deploying a simple static web app (HTML form)** on a Kubernetes cluster running on an **EC2 instance**, with **Rancher** as the cluster manager, **Docker** for containerization, and **Jenkins** for automation.

---

## Homepage URLs

- **AWS EC2 URL:** http://18.209.215.58:30080/
- **Kubernetes-deployed App URL:** http://18.209.215.58:30080/

## Video Demonstration

A video recording of the setup and working application can be found here:  
**[Video Recording Placeholder](https://gmuedu-my.sharepoint.com/:v:/g/personal/avaidya2_gmu_edu/ESILG1P2KnJHrTGJwz-AnX4Bv1p1_PWJ1qdcMs3fx5HuBw?nav=eyJyZWZlcnJhbEluZm8iOnsicmVmZXJyYWxBcHAiOiJPbmVEcml2ZUZvckJ1c2luZXNzIiwicmVmZXJyYWxBcHBQbGF0Zm9ybSI6IldlYiIsInJlZmVycmFsTW9kZSI6InZpZXciLCJyZWZlcnJhbFZpZXciOiJNeUZpbGVzTGlua0NvcHkifX0&e=6qBbtB)**

> Both URLs point to the same deployed web app via the NodePort service.

## 📋 Prerequisites
- AWS Account with permissions to create **EC2 Instances** and **Elastic IPs**.
- Basic knowledge of **Linux commands**.
- Installed tools:
  - [PuTTY / SSH client](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)  
  - [Docker Hub account](https://hub.docker.com/)  

---

## ⚡ Step-by-Step Setup

### 🔹 Step 0: Create an EC2 Instance
1. Launch an **Ubuntu EC2 instance** (recommended: `t3.large`).
   - vCPUs: 2  
   - RAM: 8 GB  
   - Storage: 30 GB  
   - OS: Ubuntu Server 22.04  
2. Configure **Security Group**:
   - Allow inbound traffic on **22 (SSH)**, **80 (HTTP)**, **443 (HTTPS)**, and **30000–32767 (NodePort range)**.  
3. Allocate and associate an **Elastic IP** to ensure a static public IP.
4. Connect via SSH (`PuTTY` or terminal).

---

### 🔹 Step 1: Install Docker
```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER
```
- Log out and back in to apply group changes.  
- Verify installation:
```bash
docker run hello-world
```

---

### 🔹 Step 2: Install Rancher
Run Rancher in a Docker container:
```bash
docker run -d --restart=unless-stopped -p 80:80 -p 443:443 --privileged rancher/rancher:latest
```
- Access Rancher UI at:  
  👉 `https://<EC2-PUBLIC-IP>`  
- Complete setup (set **admin password** etc.).

---

### 🔹 Step 3: Create Kubernetes Cluster in Rancher
1. In Rancher, create a **custom cluster**.  
2. Register your EC2 as both **Control Plane**, **Worker Node**, **etcd**.  
3. Wait until cluster shows **Active**.  
4. Download the **kubeconfig YAML** from Rancher.

---

### 🔹 Step 4: Configure `kubectl`
Install and configure `kubectl`:
```bash
sudo snap install kubectl --classic
mkdir -p ~/.kube
mv ~/kubeconfig.yaml ~/.kube/config
chmod 600 ~/.kube/config
```
Test connection:
```bash
kubectl cluster-info
kubectl get nodes -o wide
```

---

### 🔹 Step 5: Build & Push Docker Image
**Dockerfile for the static HTML app:**
```dockerfile
FROM nginx:alpine
RUN rm -rf /usr/share/nginx/html/*
COPY index.html /usr/share/nginx/html/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```
Build & push image:
```bash
docker build -t adityavaidya108/static-web-app:latest .
docker push adityavaidya108/static-web-app:latest
```

---

### 🔹 Step 6: Create Deployment & Service
**mywebapp-deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mywebapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mywebapp
  template:
    metadata:
      labels:
        app: mywebapp
    spec:
      containers:
      - name: mywebapp
        image: adityavaidya108/static-web-app:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: mywebapp-service
spec:
  selector:
    app: mywebapp
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

Apply configuration:
```bash
kubectl apply -f /home/ubuntu/mywebapp-deployment.yaml
```

Verify:
```bash
kubectl get pods
kubectl get service mywebapp-service
```

---

### 🔹 Step 7: Access the App
- Open browser:  
  👉 `http://<EC2-PUBLIC-IP>:<NODEPORT>`  
- Ensure EC2 security group allows the selected **NodePort**.

✅ You should now see your **HTML form running in 3 Pods**, managed by Kubernetes & Rancher.

---

## ⚙️ Jenkins Automation (Part 3)

Once the manual deployment works, we automate using **Jenkins**.

### Install Jenkins
```bash
sudo apt install -y openjdk-11-jdk
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt update
sudo apt install -y jenkins
sudo systemctl start jenkins
```
- Open Jenkins UI at:  
  👉 `http://<EC2-PUBLIC-IP>:8080`  
- Install plugins:
  - Docker Pipeline  
  - Kubernetes CLI  
  - Git  

---

### Jenkins CI/CD Flow
1. Create **credentials** for:
   - DockerHub  
   - GitHub  
   - Kubernetes  
2. Push project structure for GitHub:
   ```
    k8s/
    ├── deployment.yaml
    ├── service.yaml
  ```
3. Define Jenkins Pipeline:
   - Build Docker image  
   - Push to DockerHub  
   - Deploy to Kubernetes via `kubectl`  
4. Make a code change → Push to GitHub → Trigger Jenkins → Redeploy automatically

---

## Result
- **EC2 Instance** running Rancher & Kubernetes.  
- **Dockerized Web App** deployed across 3 Pods.  
- Exposed via **NodePort Service**.  
- Automated deployments via **Jenkins CI/CD**.

---

## Key Takeaways
- Docker acts as both:
  - Runtime for Kubernetes workloads  
  - Host for Rancher  
- Rancher simplifies **cluster management**.  
- Kubectl provides **developer access** to the cluster.  
- Jenkins enables **automation** for CI/CD pipelines.  

---
