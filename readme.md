# Zomato Food Delivery App — Capstone Project

> **Goal:** Deploy a Zomato clone using a fully automated CI/CD pipeline on AWS EKS with real-time monitoring.
>
> **GitHub Repo:** https://github.com/thecareerbeer/Zomato

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Required Files You Must Write](#2-required-files-you-must-write)
3. [AWS Infrastructure Setup](#3-aws-infrastructure-setup)
4. [Tool Installation on EC2](#4-tool-installation-on-ec2)
5. [Jenkins Configuration](#5-jenkins-configuration)
6. [GitHub Webhook Setup](#6-github-webhook-setup)
7. [Jenkins CI/CD Pipeline](#7-jenkins-cicd-pipeline)
8. [AWS EKS Kubernetes Deployment](#8-aws-eks-kubernetes-deployment)
9. [Monitoring — Prometheus & Grafana](#9-monitoring--prometheus--grafana)
10. [Verification & Access URLs](#10-verification--access-urls)
11. [Cleanup & Teardown](#11-cleanup--teardown)
12. [Things You Must Do Manually](#12-things-you-must-do-manually)

---

## 1. Project Overview

### Tools Required (From Capstone Doc)

| Tool | Purpose |
|---|---|
| **Git** | Version control |
| **Jenkins** | CI/CD pipeline automation |
| **Docker** | Containerize the application |
| **AWS EKS** | Kubernetes cluster for deployment |
| **AWS EC2** | Server to run Jenkins |
| **DockerHub / AWS ECR** | Store Docker images |
| **Prometheus** | Collect metrics |
| **Grafana** | Visualize dashboards |

### Pipeline Flow

```
GitHub Push
    ↓
Jenkins Webhook Trigger
    ↓
Fetch Code from GitHub
    ↓
Docker Build → Push to DockerHub
    ↓
Deploy to AWS EKS (Kubernetes)
    ↓
Prometheus + Grafana Monitoring
```

---

## 2. Required Files You Must Write

> ⚠️ The capstone says: **"Candidate needs to write Dockerfile and Kubernetes manifest."**
> These are your deliverables. Use the templates below.

### Step 1 — Fork the Repository

1. Go to: https://github.com/thecareerbeer/Zomato
2. Click **Fork** → fork to your own GitHub account
3. Clone your fork:

```bash
git clone https://github.com/<your-username>/Zomato.git
cd Zomato
```

### Step 2 — Write the Dockerfile

Create `Dockerfile` in the root of the project:

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./

RUN npm install --production

COPY . .

EXPOSE 3000

CMD ["npm", "start"]
```

### Step 3 — Write Kubernetes Manifests

Create a `kubernetes/` folder:

```bash
mkdir kubernetes
```

**`kubernetes/deployment.yml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zomato
  namespace: default
  labels:
    app: zomato
spec:
  replicas: 2
  selector:
    matchLabels:
      app: zomato
  template:
    metadata:
      labels:
        app: zomato
    spec:
      containers:
        - name: zomato
          image: <your-dockerhub-username>/zomato:latest
          ports:
            - containerPort: 3000
          resources:
            requests:
              memory: "128Mi"
              cpu: "250m"
            limits:
              memory: "256Mi"
              cpu: "500m"
```

**`kubernetes/service.yml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: zomato-service
  namespace: default
spec:
  type: NodePort
  selector:
    app: zomato
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
      nodePort: 30001
```

**`kubernetes/hpa.yml`** _(Auto-scaling — required by capstone)_

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: zomato-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: zomato
  minReplicas: 2
  maxReplicas: 6
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
```

### Step 4 — Write the Jenkinsfile

Create `Jenkinsfile` in the root of the project:

```groovy
pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node18'
    }

    environment {
        DOCKER_IMAGE = "<your-dockerhub-username>/zomato"
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Git Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/<your-username>/Zomato.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh 'docker build -t zomato .'
                    }
                }
            }
        }

        stage('Tag & Push to DockerHub') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh """
                            docker tag zomato ${DOCKER_IMAGE}:latest
                            docker push ${DOCKER_IMAGE}:latest
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    kubectl apply -f kubernetes/deployment.yml
                    kubectl apply -f kubernetes/service.yml
                    kubectl apply -f kubernetes/hpa.yml
                    kubectl rollout status deployment/zomato
                '''
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully. App deployed to EKS.'
        }
        failure {
            echo 'Pipeline failed. Check the logs above.'
        }
    }
}
```

> Replace `<your-dockerhub-username>` and `<your-username>` with your actual values.

### Step 5 — Push All Files to GitHub

```bash
git add .
git commit -m "Add Dockerfile, Kubernetes manifests, and Jenkinsfile"
git push origin main
```

---

## 3. AWS Infrastructure Setup

### Launch EC2 Instance (AWS Console)

Go to: **AWS Console → EC2 → Launch Instance**

| Setting | Value |
|---|---|
| **Name** | `zomato-server` |
| **AMI** | Ubuntu 24.04 LTS |
| **Instance Type** | t2.large |
| **Storage** | 29 GB |
| **Key Pair** | Create or select a `.pem` file |

> `t2.micro` cannot handle Jenkins + Docker. Use `t2.large` minimum.
> 29 GB keeps you within the AWS Free Tier 30 GB EBS limit.

### Security Group Inbound Rules

| Port | Purpose |
|---|---|
| 22 | SSH |
| 8080 | Jenkins |
| 3000 | Zomato App (Docker test) |

> Add port `30001` and `9100` after EKS cluster is created (for app NodePort and monitoring).

### IAM User Setup (AWS Console)

Go to: **AWS Console → IAM → Users → Create User**

- User name: `k8s`
- Attach policy: **AdministratorAccess**
- Create **Access Key** (CLI type)
- **Save the Access Key ID and Secret Access Key** — you need them in the next step

### SSH Into Your EC2

```bash
ssh -i "your-key.pem" ubuntu@<ec2-public-ip>
sudo su
sudo apt update -y
```

### Install AWS CLI

```bash
sudo apt install unzip -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Configure with your IAM credentials:

```bash
aws configure
```

Enter when prompted:
- AWS Access Key ID: `<your-access-key>`
- AWS Secret Access Key: `<your-secret-key>`
- Default region: `ap-northeast-1`
- Output format: `json`

**Verify:** `aws --version`

---

## 4. Tool Installation on EC2

### Jenkins & JDK 17

```bash
vi jenkins.sh
```

Paste the following, save with `:wq`:

```bash
#!/bin/bash
sudo apt update
sudo apt install openjdk-17-jdk -y
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt update && sudo apt install jenkins -y
sudo systemctl start jenkins
sudo systemctl enable jenkins
```

```bash
chmod +x jenkins.sh && ./jenkins.sh
```

Get the initial admin password:

```bash
cat /var/lib/jenkins/secrets/initialAdminPassword
```

Access Jenkins at: `http://<ec2-public-ip>:8080`

### Docker

```bash
vi docker.sh
```

```bash
#!/bin/bash
sudo apt install docker.io -y
sudo usermod -aG docker ubuntu
sudo usermod -aG docker jenkins
sudo systemctl restart docker
newgrp docker
```

```bash
chmod +x docker.sh && ./docker.sh
```

**Verify:** `docker --version`

Login to Docker Hub (required before pipeline runs):

```bash
docker login -u <your-dockerhub-username>
```

### kubectl

```bash
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.24.11/2023-03-17/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo cp ./kubectl /usr/local/bin
export PATH=/usr/local/bin:$PATH
kubectl version --client
```

### eksctl

```bash
curl --silent --location \
  "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
  | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

---

## 5. Jenkins Configuration

### Install Plugins

Go to: **Manage Jenkins → Plugins → Available plugins**

| Plugin | Purpose |
|---|---|
| Pipeline Stage View | Visual pipeline execution |
| Eclipse Temurin Installer | Manage Java versions |
| NodeJS | Build Node.js app |
| Docker | Docker pipeline support |
| Docker Commons | Shared Docker library |
| Docker Pipeline | Docker in Jenkinsfile DSL |
| Docker API | Docker API access |
| Docker Build Step | Docker build steps |
| Kubernetes CLI | Run kubectl in pipeline |
| Email Extension | Build notifications |

Click **Install** → check **Restart Jenkins when complete**

### Configure Global Tools

Go to: **Manage Jenkins → Tools**

> Names must exactly match what is in the Jenkinsfile.

| Tool | Exact Name | Version |
|---|---|---|
| JDK | `jdk17` | 17.0.8.1+1 (adoptium.net) |
| Node.js | `node18` | 18.x |
| Docker | `docker` | Latest (docker.com) |

### Add Credentials

Go to: **Manage Jenkins → Credentials → Global → Add Credentials**

| Kind | ID | What to Enter |
|---|---|---|
| Username with password | `docker` | Docker Hub username + password |

### Configure kubectl Access in Jenkins

After the EKS cluster is created (Section 8), run this on the EC2 to configure kubectl for Jenkins:

```bash
aws eks update-kubeconfig --name Castro-Cluster --region ap-northeast-1
# Copy the kubeconfig to Jenkins home
sudo cp ~/.kube/config /var/lib/jenkins/.kube/config
sudo chown jenkins:jenkins /var/lib/jenkins/.kube/config
```

---

## 6. GitHub Webhook Setup

This makes Jenkins trigger automatically on every `git push`.

### Step 1 — Add Webhook in GitHub

Go to your **forked repo → Settings → Webhooks → Add webhook**

| Field | Value |
|---|---|
| Payload URL | `http://<ec2-public-ip>:8080/github-webhook/` |
| Content type | `application/json` |
| Trigger | Just the **push** event |
| Active | ✅ |

Click **Add webhook**.

### Step 2 — Enable in Jenkins Job

In your Jenkins Pipeline job settings:
- **Build Triggers** → check **GitHub hook trigger for GITScm polling**

Now every push to `main` automatically triggers the pipeline.

---

## 7. Jenkins CI/CD Pipeline

### Create the Pipeline Job

1. Jenkins → **New Item**
2. Name: `zomato-pipeline` → **Pipeline** → OK
3. Under Pipeline:
   - Definition: `Pipeline script from SCM`
   - SCM: `Git`
   - Repository URL: `https://github.com/<your-username>/Zomato.git`
   - Branch: `*/main`
   - Script Path: `Jenkinsfile`
4. Check: **GitHub hook trigger for GITScm polling**
5. **Save** → **Build Now**

### Pipeline Stages

| # | Stage | What Happens | Time |
|---|---|---|---|
| 1 | Clean Workspace | Removes previous build files | Instant |
| 2 | Git Checkout | Pulls latest code from GitHub | Seconds |
| 3 | Install Dependencies | Runs `npm install` | ~1 min |
| 4 | Docker Build | Builds image as `zomato:latest` | ~2 min |
| 5 | Tag & Push | Pushes `<username>/zomato:latest` to DockerHub | ~2 min |
| 6 | Deploy to Kubernetes | Applies manifests to EKS | ~1 min |

---

## 8. AWS EKS Kubernetes Deployment

### Create EKS Cluster

```bash
eksctl create cluster \
--name cluster-1 \
--region ap-south-1 \
--node-type t3.micro \
--nodes 2
```

> ⏳ Wait approximately **20 minutes** — do not close the terminal.

### Associate IAM OIDC Provider

Required for IAM roles for service accounts — do not skip:

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster Castro-Cluster \
  --approve \
  --region ap-northeast-1
```


```
kubectl get nodes
```

Expected: Two nodes with status `Ready`.

### Deploy the App Manually (First Time)

```bash
cd Zomato
cd Kubernetes/
kubectl apply -f deployment.yml
kubectl apply -f service.yaml
kubectl apply -f hpa.yaml
kubectl get pods
kubectl get svc
```

### Verify Auto-Scaling

```bash
kubectl get hpa
```

Expected output shows `zomato-hpa` with min 2 / max 6 replicas.

### Cluster Specifications

| Setting | Value |
|---|---|
| Cluster Name | Castro-Cluster |
| Region | ap-northeast-1 (Tokyo) |
| Availability Zones | ap-northeast-1a, ap-northeast-1b |
| Node Type | t2.medium |
| Min Nodes | 2 |
| Max Nodes | 4 |
| Node Volume | 20 GB |

---

## 9. Monitoring — Prometheus & Grafana

Set up on the **same EC2 instance** or a **separate monitoring EC2** (your choice — separate is cleaner).

### Install Prometheus

```bash
# Create user and directories
sudo useradd --no-create-home --shell /bin/false prometheus
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus

# Download Prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.47.0/prometheus-2.47.0.linux-amd64.tar.gz
tar xvfz prometheus-2.47.0.linux-amd64.tar.gz
cd prometheus-2.47.0.linux-amd64

# Move binaries
sudo cp prometheus /usr/local/bin/
sudo cp promtool /usr/local/bin/
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool

# Move config
sudo cp -r consoles /etc/prometheus
sudo cp -r console_libraries /etc/prometheus
sudo chown -R prometheus:prometheus /etc/prometheus
```

### Install Node Exporter

```bash
cd ~
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
tar xvfz node_exporter-1.6.1.linux-amd64.tar.gz
sudo cp node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
sudo useradd --no-create-home --shell /bin/false nodeusr
sudo chown nodeusr:nodeusr /usr/local/bin/node_exporter
```

### Create Node Exporter Systemd Service

```bash
sudo vi /etc/systemd/system/node_exporter.service
```

```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=nodeusr
Group=nodeusr
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

### Create Prometheus Systemd Service

```bash
sudo vi /etc/systemd/system/prometheus.service
```

```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file /etc/prometheus/prometheus.yml \
  --storage.tsdb.path /var/lib/prometheus/ \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=:9090

[Install]
WantedBy=multi-user.target
```

### Configure Prometheus Scrape Jobs

```bash
sudo vi /etc/prometheus/prometheus.yml
```

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['<this-server-ip>:9100']

  - job_name: 'kubernetes_nodes'
    static_configs:
      - targets: ['<eks-node-ip>:9100']
```

> Replace `<this-server-ip>` and `<eks-node-ip>` with actual IPs.
> Add the Kubernetes target after the EKS cluster is created.

### Start All Monitoring Services

```bash
# Validate config first
promtool check config /etc/prometheus/prometheus.yml
# Expected: SUCCESS

sudo systemctl daemon-reload
sudo systemctl enable prometheus && sudo systemctl start prometheus
sudo systemctl enable node_exporter && sudo systemctl start node_exporter

# Verify
sudo systemctl status prometheus
sudo systemctl status node_exporter
```

### Install Grafana

```bash
sudo apt install -y software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | \
  sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update
sudo apt install grafana -y
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```

Access at: `http://<server-ip>:3000` — Login: `admin` / `admin`

### Connect Prometheus to Grafana (UI Steps)

1. Grafana → **Connections → Data Sources → Add data source**
2. Select **Prometheus**
3. URL: `http://<server-ip>:9090`
4. Click **Save & Test** → Should show "Data source is working"

### Import Dashboards (UI Steps)

1. Grafana → **Dashboards → Import**
2. Import ID `1860` → Load → Select Prometheus → **Import** _(Node Exporter)_

---

## 10. Verification & Access URLs

### Service URLs

| Service | URL |
|---|---|
| Jenkins | `http://<ec2-ip>:8080` |
| Zomato App (Docker test) | `http://<ec2-ip>:3000` |
| Zomato App (EKS) | `http://<eks-node-ip>:30001` |
| Prometheus | `http://<server-ip>:9090` |
| Grafana | `http://<server-ip>:3000` |

### Final Checks

```bash
# Kubernetes
kubectl get pods
kubectl get svc
kubectl get nodes
kubectl get hpa

# Docker
docker ps
docker images
```

---

## 11. Cleanup & Teardown

> ⚠️ Run after submission to avoid AWS charges.

```bash
# Delete node group first
eksctl delete nodegroup \
  --cluster=Castro-Cluster \
  --name=Castro-demo-NG

# Then delete cluster
eksctl delete cluster --name=Castro-Cluster
```

**Manual step — AWS Console:**
Go to **CloudFormation** → delete any remaining stacks starting with `eksctl-Castro-Cluster`.

**Stop EC2:**
AWS Console → EC2 → Select instance → **Instance State → Stop or Terminate**

---

## 12. Things You Must Do Manually

### 🔴 Before You Start

| # | Task | Where |
|---|---|---|
| 1 | Fork the GitHub repo | GitHub UI |
| 2 | Write the `Dockerfile` | Your editor — template in Section 2 |
| 3 | Write `kubernetes/deployment.yml`, `service.yml`, `hpa.yml` | Your editor — templates in Section 2 |
| 4 | Write the `Jenkinsfile` | Your editor — template in Section 2 |
| 5 | Create IAM user `k8s` with AdministratorAccess | AWS Console → IAM |
| 6 | Save the IAM Access Key & Secret Key | Notepad — shown only once |
| 7 | Replace all `<your-dockerhub-username>` and `<your-username>` placeholders | Jenkinsfile + deployment.yml |

### 🟡 During Setup

| # | Task | Where |
|---|---|---|
| 8 | Open Security Group ports (22, 8080, 3000, 9090, 3000, 9100, 30001) | AWS Console → EC2 → Security Groups |
| 9 | Add port 9100 on EKS Worker Node Security Group after cluster creation | AWS Console → EC2 → Security Groups |
| 10 | Add Prometheus data source in Grafana | Grafana UI |
| 11 | Import Node Exporter dashboard (ID: 1860) in Grafana | Grafana UI |

### ⏳ Steps That Take Time — Don't Close Terminal

| Step | Wait Time |
|---|---|
| EKS Cluster creation | ~20 minutes |
| Node Group creation | ~10–15 minutes |
