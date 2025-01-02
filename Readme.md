![Build Status](https://img.shields.io/badge/build-passing-brightgreen)
![License](https://img.shields.io/badge/license-MIT-blue)
![Go Version](https://img.shields.io/badge/go-1.18-blue)

# K8s Voting App

> **Note**: Works best on edge,chrome,firefox(Adjust the zoom as per your screen size), and on mobile.
> (This is a stripped down version, without source code. You can directly skip to step 2, to get the application running).

## 📚 Table of Contents
- [🏞 Overview](#-overview)
- [🔧 Prerequisites](#-prerequisites)
- [🚀 Getting Started](#-getting-started)
    - [🔨 Step 1: Build Docker Image](#-step-1-build-docker-image)
    - [🛠 Step 2: Deploy to Kubernetes](#-step-2-deploy-to-kubernetes)
    - [🧹 Step 3: Clean Up](#-step-3-clean-up-existing-deployments-optional)
    - [📈 Step 4: Monitor](#-step-4-monitor-pods-and-hpa)
    - [📊 Step 5: Metrics Server](#-step-5-metrics-server-setup)
    - [🛡️ Step 6: Configure Firewall](#-step-6-configure-firewall)
- [🌐 Access Points](#-access-points)
- [🧪 Testing](#-testing)
- [🔄 Rolling Updates](#-rolling-updates)
- [📊 Monitoring](#-monitoring)
- [📄 License](#-license)

## 🏞 Overview
This repository contains a **Voting Application** built with [Go](https://golang.org/), designed to run on [Kubernetes](https://kubernetes.io/). It utilizes [Redis](https://redis.io/) for data storage and includes a Horizontal Pod Autoscaler (HPA) to manage scaling based on CPU utilization. The application can be accessed locally and also over a local network through specified ports.

## 🔧 Prerequisites
- **[Docker Desktop](https://www.docker.com/products/docker-desktop)**: Ensure Docker is installed and running on your machine.
- **[Kubernetes](https://kubernetes.io/)**: A Kubernetes cluster (using Docker Desktop as the Kubernetes context).
- **[kubectl](https://kubernetes.io/docs/tasks/tools/)**: Command-line tool for interacting with the Kubernetes cluster.
- **[Metrics Server](https://github.com/kubernetes-sigs/metrics-server)**: Installed in your Kubernetes cluster to enable HPA functionality.

## 🚀 Getting Started
Enable Kubernetes in Docker Desktop.

### 🔨 Step 1: Build the Docker Image
Build the Docker image for the voting application:
```bash
docker build -t voting-app:latest .
```
*Or directly proceed to Step 2 if you don't want to build your own image.*

### 🛠 Step 2: Deploy to Kubernetes
Apply the Kubernetes configurations:
```bash
kubectl apply -f k8s/
```
*Apply again if there's any error.*

### 🧹 Step 3: Clean Up Existing Deployments (Optional)
To delete existing deployments and services:
```bash
kubectl delete deployments --all
kubectl delete services --all
kubectl delete deployments --all -n monitoring
kubectl delete services --all -n monitoring
```

### 📈 Step 4: Monitor Pods and HPA

**Terminal 1:**
Check pods and HPA status:
```bash
kubectl get hpa voting-app-hpa --watch
```

**Terminal 2:**
```bash
kubectl get pods --selector=app=voting-app --watch
```

### 📊 Step 5: Metrics Server Setup
> **Note**: If Metrics Server is not already running in your cluster, follow these steps to set it up.
>
> **Update**: The `components.yaml` for Metrics Server is now included in the `k8s` folder with required configurations. You can skip to Step 6.

1. **Download the components YAML:**
    ```bash
    kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
    ```

2. **Modify the components YAML**:
    - Add the following lines under `spec.template.spec.containers.args`:
    ```yaml
        - --kubelet-insecure-tls
    ```

3. **Apply the configuration:**
    ```bash
    kubectl apply -f components.yaml
    ```

4. **Verify Metrics Server is running:**
    ```bash
    kubectl get deployment metrics-server -n kube-system
    kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes
    ```

> **Reminder**: It may take some time for the Metrics API to become available. Please be patient.

5. **Ensure everything is working:**
    ```bash
    kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes
    
    kubectl get deployment metrics-server -n kube-system
    ```


*Great! Your Metrics Server should be up and running now. 🚀*

### 🛡️ Step 6: Configure Firewall
Allow traffic on port `30080`:

**For Linux:**
```bash
sudo ufw allow 30080/tcp
```

**For Windows (PowerShell - Admin Mode):**
```powershell
netsh advfirewall firewall add rule name="Allow Port 30080" dir=in action=allow protocol=TCP localport=30080
```

## 🌐 Access Points
- **Application**: `http://localhost:30080`
- **RedisInsight**: `http://localhost:30081`

- For devices on local network:
  http:"your-ipv4-address":30080 
// remove the commas.

### 🔑 Command to Set Admin Username and Password in Redis
In RedisInsight CLI:
```bash
HSET admins <admin_username> <admin_password>
```
*For example:*
```bash
HSET admins admin admin123
```
This means:
AdminUsername: admin
AdminPassword: admin123

You can customize as per your liking :)

*This allows you to log in as admin and create custom votings!*

- **Redis Connection URL**: `redis://default@redis:6379`

## 🧪 Testing
**Install `hey`:**
```bash
sudo apt install hey
```

**Run Load Test:**
```bash
hey -z 10m -c 6000 -q 2000 http://localhost:30080/
```

**Test the Application:**
```bash
curl http://localhost:30080
curl http://192.168.1.209:30080  # Replace with your system's network address
```

## 🔄 Rolling Updates

1. **Build the Latest Image:**
    ```bash
    docker build -t voting-app:latest .
    ```

2. **Update the Image in Deployment:**
    - Edit the image source in your Kubernetes deployment to use the latest image tag.

    ![spec.containers.image](misc/appdeployement.png) 

3. **Apply Changes:**
    ```bash
    kubectl apply -f k8s/
    ```

###  Watch the Rollout Status
```bash
kubectl rollout status deployment/voting-app
```

### 🐳 Check the Pods
```bash
kubectl get pods
```

### 📋 Check Deployment Details (If Needed)
```bash
kubectl describe deployment voting-app
```

*Watch the update occur within a few minutes without any downtime!*

## 📊 Monitoring

### 📈 Accessing Grafana
1. **Open Browser:** Navigate to `http://localhost:30091`.
2. **Login Credentials:**
    - **Username:** `admin`
    - **Password:** `admin` (You will be prompted to change this after the first login)

### 📊 Adding Prometheus as a Data Source in Grafana
1. **Navigate:** Go to `Configuration` > `Data Sources`.
2. **Add Data Source:** Click `Add data source`.
3. **Select Prometheus:** Choose `Prometheus` from the list.
4. **Set URL:** Enter `http://prometheus:9090`.
5. **Verify Connection:** Click `Save & Test` to ensure the connection is successful.

### 📈 Creating Dashboards
1. **Create Dashboard:** Go to `Create` > `Dashboard`.
2. **Add Panel:** Click `Add new panel`.
3. **Select Data Source:** Choose Prometheus and enter your query.
4. **Customize:** Adjust your panel settings as desired.
5. **Apply Changes:** Click `Apply`.

**Check Available Metrics:**
```bash
kubectl exec -it $(kubectl get pods -n monitoring | grep prometheus | awk '{print $1}') -n monitoring -- wget -qO- voting-app-service.default:8080/metrics
```

*You can now monitor your Prometheus metrics through Grafana dashboards.*

