# Voting App

## Table of Contents
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
    - [Step 1: Build Docker Image](#step-1-build-the-docker-image)
    - [Step 2: Deploy to Kubernetes](#step-2-deploy-to-kubernetes)
    - [Step 3: Clean Up](#step-3-clean-up-existing-deployments-optional)
    - [Step 4: Monitor](#step-4-monitor-pods-and-hpa)
    - [Step 5: Metrics Server](#step-5-setting-up-your-metrics-server)
    - [Step 6: Configure Firewall](#step-6-configure-firewall)
- [Access Points](#access-points)
- [Testing](#testing)
- [Rolling Updates](#to-roll-out-an-update)
- [Monitoring](#monitoring)

## Overview
This repository contains a voting application built with [Go](https://golang.org/), designed to run on [Kubernetes](https://kubernetes.io/). It utilizes [Redis](https://redis.io/) for data storage and includes a Horizontal Pod Autoscaler (HPA) to manage scaling based on CPU utilization. The application can be accessed locally and also over local network through specified ports.

## Prerequisites
- **[Docker Desktop](https://www.docker.com/products/docker-desktop)**: Ensure Docker is installed and running on your machine.
- **[Kubernetes](https://kubernetes.io/)**: A Kubernetes cluster, in this case, we are using docker-desktop as our      kubernetescontext
- **[kubectl](https://kubernetes.io/docs/tasks/tools/)**: Command-line tool for interacting with the Kubernetes cluster.
- **[Metrics Server](https://github.com/kubernetes-sigs/metrics-server)**: Installed in your Kubernetes cluster to enable HPA functionality.

## Getting Started
Enale kubernetes in docker desktop.

### Step 1: In your bash shell/powershell:
Build the Docker image for the voting application:
```bash
kubectl apply -f .
```

### Step 2: Deploy to Kubernetes
Apply the Kubernetes configurations:
```bash
kubectl apply -f k8s/
```
Apply again, if there's any error.


### Step 3: Clean Up Existing Deployments (Optional)
To delete existing deployments and services:
```bash
kubectl delete deployments --all
kubectl delete services --all
kubectl delete deployments --all -n monitoring
kubectl delete services --all -n monitoring
```

### Step 4: Monitor Pods and HPA
Terminal 1
Check pods and HPA status:
```bash
kubectl get hpa voting-app-hpa --watch
```
Terminal 2
```bash
kubectl get pods --selector=app=voting-app --watch
```
### Step 5: Setting Up Your Metrics Server

Don't worry if you're seeing `<unknown>` in your Targets section - this is normal with Docker Desktop! Let's fix that:

1. First, let's check if your metrics are working:
```bash
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes
```

2. Not working? Docker Desktop doesn't ship with Kubernetes metrics by default. Let's install the metrics server from the [official Kubernetes metrics-server repository](https://github.com/kubernetes-sigs/metrics-server#installation):
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

3. Now we need to make a quick tweak. Open the metrics server deployment:
```bash
kubectl edit deployment metrics-server -n kube-system
```

4. Here's what to do in the editor:
    - Hit `i` to start editing
    - Find the `args` section under `spec.containers`
    - Add this line: `- --kubelet-insecure-tls`
    - Press `Esc` and type `:wq` to save and exit

5. ## Configuration Overview
![Final Configuration](misc/final-config.png)

The above diagram shows the complete configuration of our voting application deployment.

5. Let's make sure everything's working now:
```bash
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes
```

6. Apply, clean up your existing deployments:
```bash
kubectl delete deployments --all
kubectl delete services --all
```

Great! Your metrics server should be up and running now. 🚀


### Step 6: Configure Firewall
Allow traffic on port 30080:

For linux
```bash
sudo ufw allow 30080/tcp
```
For windows: In powershell(Admin mode)
```powershell
netsh advfirewall firewall add rule name="Allow Port 30080" dir=in action=allow protocol=TCP localport=30080
```

## Access Points
- Application: `http://localhost:30080`
- RedisInsight: `http://localhost:30081`
- Redis Connection URL: `redis://default@redis:6379`

## Testing
Install hey:
```bash
sudo apt install hey
```

```bash
hey -z 10m -c 6000 -q 2000 http://localhost:30080/
```
Test the application using:
```bash
curl http://localhost:30080
curl http://192.168.1.209:30080  # Replace with your systems network address
```
## To roll out an update

1. Build the latest image
2. In image source, use the latest image tag that you have used while building the image

![spec.containers.image](misc/appdeployement.png) 

Update the image in spec.containers.image

3. Apply changes

```bash
kubectl apply -f k8s/
```

# Watch the rollout status
```bash
kubectl rollout status deployment/voting-app
```

# Check the pods
```bash
kubectl get pods
```

# If you need to check the deployment details
```bash
kubectl describe deployment voting-app
```

Watch the update happen within a few minutes without any downtime!

## Monitoring
### Accessing Grafana
1. Open your browser and navigate to `http://localhost:30091`.
2. Login with the default credentials:
   - Username: `admin`
   - Password: `admin` (you will be prompted to change this after the first login)

### Adding Prometheus as a Data Source in Grafana
1. In Grafana, go to `Configuration` > `Data Sources`.
2. Click `Add data source`.
3. Select `Prometheus`.
4. Set the URL to `http://prometheus:9090`.
5. Click `Save & Test` to verify the connection.

### Creating Dashboards
1. Go to `Create` > `Dashboard`.
2. Click `Add new panel`.
3. Select the Prometheus data source and enter your query.
4. Customize your panel and click `Apply`.
5. To check the available metrics, in bash
```bash
kubectl exec -it $(kubectl get pods -n monitoring | grep prometheus | awk '{print $1}') -n monitoring -- wget -qO- voting-app-service.default:8080/metrics
```


You can now monitor your Prometheus metrics through Grafana dashboards.
