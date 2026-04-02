# Zomato Food Delivery App — Capstone DevOps Project

> **Capstone Objective:** Deploy a Zomato clone on AWS EKS with a fully automated CI/CD pipeline using Jenkins, SonarQube, OWASP, Trivy, Docker Scout, Prometheus, Grafana, and Argo CD.
>
> **GitHub Repo:** https://github.com/thecareerbeer/Zomato
>
> ⚠️ **Important:** The capstone requires you to write your own `Dockerfile` and Kubernetes manifests. Templates are provided in this document.

---

## Table of Contents

1. [Project Overview & Business Goals](#1-project-overview--business-goals)
2. [Prerequisites & Environment Setup](#2-prerequisites--environment-setup)
3. [Fork Repo & Write Required Files](#3-fork-repo--write-required-files)
4. [Tool Installation on Zomato Server](#4-tool-installation-on-zomato-server)
5. [Jenkins Configuration](#5-jenkins-configuration)
6. [GitHub Webhook Setup](#6-github-webhook-setup)
7. [Phase 1 — CI/CD Pipeline (Docker Deployment)](#7-phase-1--cicd-pipeline-docker-deployment)
8. [Service Access URLs](#8-service-access-urls)
9. [Monitoring Stack — Prometheus, Node Exporter & Grafana](#9-monitoring-stack--prometheus-node-exporter--grafana)
10. [Phase 2 — AWS EKS Kubernetes Deployment](#10-phase-2--aws-eks-kubernetes-deployment)
11. [Argo CD (GitOps) Setup](#11-argo-cd-gitops-setup)
12. [Final Deployment & Verification](#12-final-deployment--verification)
13. [Troubleshooting](#13-troubleshooting)
14. [Cleanup & Teardown](#14-cleanup--teardown)
15. [Things You Must Do Manually](#15-things-you-must-do-manually)

---

## 1. Project Overview & Business Goals

### What You Are Building

A fully automated DevOps pipeline for a **Zomato food delivery clone** that:
- Automatically builds and tests code on every GitHub push
- Scans for code quality issues, dependency vulnerabilities, and image vulnerabilities
- Deploys to a Docker container (Phase 1) and then to AWS EKS (Phase 2)
- Monitors everything with Prometheus and Grafana

### Business Goals (From Capstone Doc)

- Faster and reliable deployments — zero manual deployments
- Reduced manual intervention in infrastructure management
- Improved scalability via Kubernetes auto-scaling
- Real-time monitoring and alerting
- Automated rollback in case of failures

### Pipeline Flow

```
GitHub Push → Jenkins Webhook Trigger → Code Checkout
→ SonarQube Analysis → Quality Gate
→ OWASP Scan → Trivy FS Scan
→ Docker Build → Tag & Push to Docker Hub
→ Docker Scout Analysis
→ Deploy to Docker Container (Phase 1 / Staging)
→ Deploy to AWS EKS via Argo CD (Phase 2 / Production)
→ Prometheus + Grafana Monitoring
```

### Technical Toolset

| Category | Tools |
|---|---|
| **Source Control** | GitHub |
| **CI/CD** | Jenkins |
| **Code Quality** | SonarQube (LTS Community) |
| **Security** | OWASP Dependency-Check, Trivy, Docker Scout |
| **Containerization** | Docker, Docker Hub |
| **Orchestration** | AWS EKS, kubectl, eksctl |
| **GitOps** | Argo CD |
| **Monitoring** | Prometheus, Node Exporter, Grafana |
| **Notifications** | Jenkins Email Extension (SMTP) |

---

## 2. Prerequisites & Environment Setup

### Step 1 — Launch Two EC2 Instances (AWS Console)

Go to: **AWS Console → EC2 → Launch Instance**

| Setting | Zomato Server | Monitoring Server |
|---|---|---|
| **Name** | `zomato-server` | `monitoring-server` |
| **AMI** | Ubuntu 24.04 LTS | Ubuntu 24.04 LTS |
| **Instance Type** | t2.large | t2.large |
| **Storage** | 29 GB | 29 GB |
| **Key Pair** | Create/select `.pem` file | Same `.pem` file |

> `t2.micro` cannot handle all the tools. Always use `t2.large`.
> 29 GB keeps you within the AWS Free Tier 30 GB EBS limit.

### Step 2 — Configure Security Group Inbound Rules

For the **Zomato Server**:

| Port | Protocol | Purpose |
|---|---|---|
| 22 | TCP | SSH |
| 8080 | TCP | Jenkins |
| 9000 | TCP | SonarQube |
| 3000 | TCP | Zomato App (Docker) |
| 9100 | TCP | Node Exporter |

For the **Monitoring Server**:

| Port | Protocol | Purpose |
|---|---|---|
| 22 | TCP | SSH |
| 9090 | TCP | Prometheus |
| 3000 | TCP | Grafana |
| 9100 | TCP | Node Exporter |

For **EKS Worker Nodes** (add after cluster creation):

| Port | Protocol | Purpose |
|---|---|---|
| 9100 | TCP | Node Exporter |
| 30001 | TCP | Zomato App NodePort |

### Step 3 — IAM Setup (AWS Console)

Go to: **AWS Console → IAM → Users → Create User**

- User name: `k8s`
- Attach policy: **AdministratorAccess**
- Create **Access Key** (CLI type) → Save the `Access Key ID` and `Secret Access Key`

### Step 4 — SSH Into Zomato Server

```bash
ssh -i "your-key.pem" ubuntu@<zomato-server-public-ip>
sudo su
sudo apt update -y
```

### Step 5 — Install AWS CLI

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
- Default region name: `ap-northeast-1`
- Default output format: `json`

**Verify:** `aws --version` → Expected: `2.18.74`

---

## 3. Fork Repo & Write Required Files

> ⚠️ **Capstone Requirement:** You must write the `Dockerfile` and Kubernetes manifests yourself. Use the templates below.

### Step 1 — Fork the Repository

1. Go to: https://github.com/thecareerbeer/Zomato
2. Click **Fork** → Fork to your own GitHub account
3. Clone your fork to the Zomato Server:

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

Create a folder called `kubernetes/` in your repo:

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

Create `Jenkinsfile` in the root of your repo:

```groovy
pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node23'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
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
                git branch: 'main', url: 'https://github.com/<your-username>/Zomato.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=zomato \
                        -Dsonar.projectKey=zomato
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false,
                        credentialsId: 'sonar-token'
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('OWASP Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit',
                    odcInstallation: 'dp-check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs . > trivy.txt'
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

        stage('Tag & Push to Docker Hub') {
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

        stage('Docker Scout') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh 'docker scout quickview ${DOCKER_IMAGE}:latest'
                        sh 'docker scout cves ${DOCKER_IMAGE}:latest'
                    }
                }
            }
        }

        stage('Deploy to Container') {
            steps {
                sh 'docker run -d --name zomato -p 3000:3000 ${DOCKER_IMAGE}:latest'
            }
        }
    }

    post {
        always {
            emailext(
                attachLog: true,
                subject: "'${currentBuild.result}'",
                body: "Project: ${env.JOB_NAME}<br/>Build Number: ${env.BUILD_NUMBER}<br/>URL: ${env.BUILD_URL}<br/>",
                to: '<your-email@gmail.com>',
                attachmentsPattern: 'trivy.txt'
            )
        }
    }
}
```

> Replace `<your-dockerhub-username>`, `<your-username>`, and `<your-email@gmail.com>` with your actual values.

### Step 5 — Push All Files to GitHub

```bash
git add .
git commit -m "Add Dockerfile, Kubernetes manifests, and Jenkinsfile"
git push origin main
```

---

## 4. Tool Installation on Zomato Server

### Jenkins & JDK 17 — `jenkins.sh`

```bash
vi jenkins.sh
```

Paste the following, then save (`:wq`):

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

### Docker — `docker.sh`

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

**Verify:** `docker --version` → Expected: `27`

### Trivy — `trivy.sh`

```bash
vi trivy.sh
```

```bash
#!/bin/bash
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | \
  sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt update && sudo apt install trivy -y
```

```bash
chmod +x trivy.sh && ./trivy.sh
```

**Verify:** `trivy --version` → Expected: `0.56.2`

### Docker Scout

```bash
# Login to Docker Hub first (required before Scout works)
docker login -u <your-dockerhub-username>

# Install Docker Scout CLI
curl -sSfL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh | sh -s --
sudo chmod +x /usr/local/bin/docker-scout
```

### SonarQube

```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

Access at: `http://<zomato-server-ip>:9000`
- Default login: `admin` / `admin`
- You will be forced to change the password on first login

**Generate SonarQube Token:**
`Administration → Security → Users → Update Tokens → Generate`
- Token name: `token`
- Expiration: 30 days
- **Copy and save the token immediately** — it only shows once

---

## 5. Jenkins Configuration

### Install Plugins

Go to: **Manage Jenkins → Plugins → Available plugins**

Search and install each:

| Plugin Name | Purpose |
|---|---|
| Pipeline Stage View | Visual stage execution |
| Eclipse Temurin Installer | Manage Java versions |
| SonarQube Scanner | SonarQube integration |
| Docker | Docker pipeline support |
| Docker Commons | Docker shared library |
| Docker Pipeline | Docker in pipeline DSL |
| Docker API | Docker API access |
| Docker Build Step | Docker build steps |
| Email Extension Template | HTML email notifications |
| OWASP Dependency-Check | Vulnerability scanning |
| NodeJS | Node.js build support |
| Prometheus Metrics | Expose Jenkins metrics |

Click **Install** → Check **Restart Jenkins when installation is complete**

### Configure Global Tools

Go to: **Manage Jenkins → Tools**

> ⚠️ Names must exactly match the Jenkinsfile or builds will fail.

| Tool | Name (exact) | Version |
|---|---|---|
| JDK | `jdk17` | 17.0.8.1+1 (from adoptium.net) |
| SonarQube Scanner | `sonar-scanner` | 6.2.1.4610 |
| Node.js | `node23` | 23.10 |
| Dependency-Check | `dp-check` | 11.1.0 (GitHub releases) |
| Docker | `docker` | Latest (docker.com) |

### Add Credentials

Go to: **Manage Jenkins → Credentials → Global → Add Credentials**

| # | Kind | ID | What to Enter |
|---|---|---|---|
| 1 | Secret text | `sonar-token` | The token generated from SonarQube |
| 2 | Username with password | `docker` | Docker Hub username + password |
| 3 | Username with password | `email-cred` | Gmail address + Google App Password |

**To generate Gmail App Password:**
Google Account → Security → 2-Step Verification → App passwords → Generate for "Mail"

### Configure Jenkins System Settings

Go to: **Manage Jenkins → System**

**SonarQube Server:**
- Name: `sonar-server`
- Server URL: `http://<zomato-server-ip>:9000`
- Server authentication token: select `sonar-token`

**Extended E-mail Notification:**
- SMTP server: `smtp.gmail.com`
- SMTP Port: `465`
- Advanced → Credentials: `email-cred`
- Enable SSL: ✅
- Default Triggers: `Always` + `Failure` + `Success`

**SonarQube Webhook** (set inside SonarQube UI, not Jenkins):
`SonarQube → Administration → Configuration → Webhooks → Create`
- Name: `Jenkins`
- URL: `http://<zomato-server-ip>:8080/sonarqube-webhook/`

---

## 6. GitHub Webhook Setup

This makes Jenkins trigger automatically on every `git push`.

### Step 1 — Get Jenkins URL

Your webhook URL will be:
```
http://<zomato-server-ip>:8080/github-webhook/
```

### Step 2 — Add Webhook in GitHub

Go to your **forked GitHub repo** → **Settings → Webhooks → Add webhook**

| Field | Value |
|---|---|
| Payload URL | `http://<zomato-server-ip>:8080/github-webhook/` |
| Content type | `application/json` |
| Which events | **Just the push event** |
| Active | ✅ checked |

Click **Add webhook**.

### Step 3 — Configure Jenkins Job for Webhook

When creating your Jenkins Pipeline job:
- Under **Build Triggers** → check **GitHub hook trigger for GITScm polling**

Now every push to the `main` branch will auto-trigger the pipeline.

---

## 7. Phase 1 — CI/CD Pipeline (Docker Deployment)

### Create the Jenkins Pipeline Job

1. Jenkins Dashboard → **New Item**
2. Name: `zomato-pipeline`
3. Type: **Pipeline** → OK
4. Under **Pipeline**:
   - Definition: `Pipeline script from SCM`
   - SCM: `Git`
   - Repository URL: `https://github.com/<your-username>/Zomato.git`
   - Branch: `*/main`
   - Script Path: `Jenkinsfile`
5. Check: **GitHub hook trigger for GITScm polling**
6. Click **Save** → Click **Build Now**

### Pipeline Stages Summary

| # | Stage | What Happens | Wait Time |
|---|---|---|---|
| 1 | Clean Workspace | Wipes previous build files | Instant |
| 2 | Git Checkout | Pulls latest code from GitHub | Seconds |
| 3 | SonarQube Analysis | Scans code quality | ~2 min |
| 4 | Quality Gate | Passes/fails based on Sonar result | ~1 min |
| 5 | Install Dependencies | `npm install` | ~1 min |
| 6 | OWASP Scan | Dependency vulnerability scan | ⏳ ~30 min |
| 7 | Trivy FS Scan | Filesystem vulnerability scan | ~2 min |
| 8 | Docker Build | Builds image locally | ~2 min |
| 9 | Tag & Push | Pushes to Docker Hub | ~2 min |
| 10 | Docker Scout | Vulnerability analysis on pushed image | ~1 min |
| 11 | Deploy to Container | Runs app on port 3000 | Instant |
| 12 | Email Notification | Sends result + trivy.txt | Instant |

> ⚠️ OWASP scan will show as "unstable" (yellow) on first run — this is normal. The pipeline continues.

---

## 8. Service Access URLs

Bookmark these — you will need them throughout the project.

| Service | URL | Login |
|---|---|---|
| Jenkins | `http://<zomato-server-ip>:8080` | admin / (initial password) |
| SonarQube | `http://<zomato-server-ip>:9000` | admin / (your password) |
| Zomato App (Docker) | `http://<zomato-server-ip>:3000` | None |
| Prometheus | `http://<monitoring-server-ip>:9090` | None |
| Grafana | `http://<monitoring-server-ip>:3000` | admin / admin |
| Zomato App (EKS) | `http://<eks-node-ip>:30001` | None |

---

## 9. Monitoring Stack — Prometheus, Node Exporter & Grafana

SSH into the **Monitoring Server**:

```bash
ssh -i "your-key.pem" ubuntu@<monitoring-server-ip>
sudo su
sudo apt update -y
```

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

# Move config files
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

### Create Node Exporter Service

```bash
sudo vi /etc/systemd/system/node_exporter.service
```

Paste:

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

### Create Prometheus Service

```bash
sudo vi /etc/systemd/system/prometheus.service
```

Paste:

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

Replace/append the `scrape_configs` section:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['<monitoring-server-ip>:9100']

  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['<zomato-server-ip>:8080']

  - job_name: 'kubernetes'
    static_configs:
      - targets: ['<eks-node-ip>:9100']
```

> Replace `<monitoring-server-ip>`, `<zomato-server-ip>`, and `<eks-node-ip>` with your actual IPs.
> Add the Kubernetes target after the EKS cluster is created.

### Validate and Start Services

```bash
# Validate prometheus config
promtool check config /etc/prometheus/prometheus.yml
# Expected: SUCCESS

# Start all services
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
  sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt update
sudo apt install grafana -y
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```

Access Grafana at: `http://<monitoring-server-ip>:3000`
- Default login: `admin` / `admin` (you will be asked to change it)

### Connect Prometheus to Grafana

1. Grafana UI → **Connections → Data Sources → Add data source**
2. Select **Prometheus**
3. URL: `http://<monitoring-server-ip>:9090`
4. Click **Save & Test** → Should show "Data source is working"

### Import Jenkins Dashboard

1. Grafana UI → **Dashboards → Import**
2. Dashboard ID: `9964` → Click **Load**
3. Select Prometheus as data source → **Import**

---

## 10. Phase 2 — AWS EKS Kubernetes Deployment

### Install eksctl

```bash
curl --silent --location \
  "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
  | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

### Install kubectl

```bash
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.24.11/2023-03-17/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo cp ./kubectl /usr/local/bin
export PATH=/usr/local/bin:$PATH
kubectl version --client
```

### Create EKS Cluster

```bash
eksctl create cluster \
  --name Castro-Cluster \
  --region ap-northeast-1 \
  --zones ap-northeast-1a,ap-northeast-1b
```

> ⏳ Wait approximately **20 minutes** — do not close the terminal.

### Associate IAM OIDC Provider

Required for IAM roles for Kubernetes service accounts (do not skip):

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster Castro-Cluster \
  --approve \
  --region ap-northeast-1
```

### Create Node Group

```bash
eksctl create nodegroup \
  --cluster=Castro-Cluster \
  --region=ap-northeast-1 \
  --name=Castro-demo-NG \
  --node-type=t2.medium \
  --nodes=2 \
  --nodes-min=2 \
  --nodes-max=4 \
  --node-volume-size=20 \
  --ssh-access \
  --ssh-public-key=<your-pem-key-name>
```

> ⏳ Wait approximately **10–15 minutes**.

### Connect kubectl to Cluster

```bash
aws eks update-kubeconfig --name Castro-Cluster --region ap-northeast-1
kubectl get nodes
```

Expected output: Two nodes with status `Ready`.

### Apply Auto-Scaling (HPA)

After deploying the app via Argo CD:

```bash
kubectl apply -f kubernetes/hpa.yml
kubectl get hpa
```

### EKS Cluster Specifications

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

## 11. Argo CD (GitOps) Setup

> ⚠️ **Critical:** The `kubectl patch` command must be run in **Windows Command Prompt (Admin)** or **PowerShell** — it regularly fails in VS Code terminals.

### Install Argo CD

```bash
# Step 1: Create namespace
kubectl create namespace argocd

# Step 2: Install Argo CD
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Step 3: Expose Argo CD via LoadBalancer
kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "LoadBalancer"}}'
```

> ⏳ Wait ~5 minutes for the LoadBalancer URL to be provisioned.

```bash
# Get the LoadBalancer URL
kubectl get svc argocd-server -n argocd
```

### Get Argo CD Initial Admin Password

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd -o jsonpath="{.data.password}" | base64 -d
```

### Verify Argo CD Is Running

```bash
kubectl get pods -n argocd
kubectl get namespace
```

All pods should show `Running`.

### Create Argo CD Application

1. Open Argo CD UI using the LoadBalancer URL in your browser
2. Login: username `admin`, password from above command
3. Click **+ New App**

| Field | Value |
|---|---|
| Application Name | `zomato` _(must be lowercase)_ |
| Project | `default` |
| Sync Policy | `Automatic` |
| Repository URL | `https://github.com/<your-username>/Zomato.git` |
| Revision | `main` |
| Path | `kubernetes` |
| Cluster URL | `https://kubernetes.default.svc` |
| Namespace | `default` |

4. Click **Create** → Click **Sync** → Click **Synchronize**

---

## 12. Final Deployment & Verification

### Update deployment.yml Before Syncing

In your GitHub repo, make sure `kubernetes/deployment.yml` has your actual Docker Hub image:

```yaml
image: <your-dockerhub-username>/zomato:latest
```

Commit and push this change before syncing in Argo CD.

### Verify Everything Is Running

```bash
# Check pods
kubectl get pods
kubectl get pods -n argocd

# Check services
kubectl get svc

# Check nodes
kubectl get nodes

# Check HPA
kubectl get hpa

# Check Docker on Zomato Server
docker ps
docker images
```

### Final Access Test

| What | URL | Should Show |
|---|---|---|
| Jenkins | `http://<zomato-server-ip>:8080` | Jenkins dashboard |
| SonarQube | `http://<zomato-server-ip>:9000` | Zomato project quality report |
| App (Docker) | `http://<zomato-server-ip>:3000` | Zomato UI |
| Prometheus | `http://<monitoring-server-ip>:9090` | Prometheus targets (all green) |
| Grafana | `http://<monitoring-server-ip>:3000` | Dashboards with live metrics |
| App (EKS) | `http://<eks-node-ip>:30001` | Zomato UI from Kubernetes |

---

## 13. Troubleshooting

### Port 3000 Already in Use (Pipeline Re-run)

**Error:** "Deploy to Container" fails — port 3000 already bound.

**Fix:**
```bash
docker stop zomato && docker rm zomato
```

Re-run the pipeline.

### OWASP Build Shows "Unstable" (Yellow)

**Cause:** High-severity vulnerabilities found or scan timed out.

**Fix:** Normal behaviour — the pipeline continues. You can adjust the Jenkinsfile to ignore the unstable status.

### Prometheus Shows Node Exporter 0/1 Up

**Cause:** Port 9100 not open on EKS Worker Node Security Group.

**Fix:** AWS Console → EC2 → Security Groups → Find the Node Group SG → Add inbound rule: TCP port `9100`.

### Argo CD App Creation Fails — "Invalid Metadata"

**Cause:** Uppercase letters in the app name.

**Fix:** Use all lowercase: `zomato` not `Zomato`.

### kubectl Commands Fail in VS Code Terminal

**Fix:** Switch to **Windows Command Prompt** or **PowerShell (Admin)** and run from there.

### Jenkins Cannot Connect to Docker

**Cause:** Jenkins user not in Docker group.

**Fix:**
```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

### Prometheus Config Validation Fails

**Fix:**
```bash
promtool check config /etc/prometheus/prometheus.yml
```
Check for YAML indentation errors — YAML is whitespace-sensitive.

### Expected Version Outputs

| Command | Expected Output |
|---|---|
| `aws --version` | `2.18.74` |
| `docker --version` | `27` |
| `trivy --version` | `0.56.2` |
| `promtool check config ...` | `SUCCESS` |
| `eksctl version` | Latest |
| `kubectl version --client` | Matches EKS version |

---

## 14. Cleanup & Teardown

> ⚠️ Do this after submission to avoid ongoing AWS charges.

```bash
# Step 1: Delete Node Group
eksctl delete nodegroup \
  --cluster=Castro-Cluster \
  --name=Castro-demo-NG

# Step 2: Delete Cluster
eksctl delete cluster --name=Castro-Cluster

# Step 3: Stop Docker containers on Zomato Server
docker stop zomato sonar
docker rm zomato sonar

# Step 4: Stop EC2 instances (AWS Console)
# Go to EC2 → Instances → Select both → Instance State → Stop/Terminate
```

**Step 5 — Manual CloudFormation Cleanup:**
AWS Console → **CloudFormation** → Delete any remaining stacks with names starting with `eksctl-Castro-Cluster`.

---

## 15. Things You Must Do Manually

These steps **cannot be scripted or automated** — you must do them yourself carefully:

### 🔴 Critical — Must Do Before Anything Else

| # | Task | Where | Why |
|---|---|---|---|
| 1 | **Fork the GitHub repo** | GitHub UI | You need your own copy to push Dockerfile, Jenkinsfile, and manifests |
| 2 | **Write Dockerfile** | Your code editor | Capstone requirement — templates provided in Section 3 |
| 3 | **Write Kubernetes manifests** | Your code editor | Capstone requirement — templates provided in Section 3 |
| 4 | **Create IAM user `k8s`** | AWS Console → IAM | Required for EKS and AWS CLI access |
| 5 | **Save IAM Access Key & Secret** | Notepad/secure place | You only see the secret key once — if lost, create a new one |

### 🟡 Must Do During Setup

| # | Task | Where | Why |
|---|---|---|---|
| 6 | **Open Security Group ports** | AWS Console → EC2 → Security Groups | Wrong/missing ports = services unreachable |
| 7 | **Generate SonarQube token** | SonarQube UI → Admin → Security | Token only shown once — copy it immediately |
| 8 | **Generate Gmail App Password** | Google Account → Security | Regular Gmail password does not work with SMTP |
| 9 | **Set up GitHub Webhook** | GitHub repo → Settings → Webhooks | Without this, Jenkins won't auto-trigger on push |
| 10 | **Replace all placeholder values** in Jenkinsfile and `deployment.yml` | Code editor | `<your-dockerhub-username>` etc. must be your real values |

### 🟠 Must Do During Kubernetes Phase

| # | Task | Where | Why |
|---|---|---|---|
| 11 | **Run `kubectl patch` in PowerShell/CMD** | Windows terminal | Fails silently in VS Code terminal |
| 12 | **Open port 9100 on EKS Node Group SG** | AWS Console → EC2 → Security Groups | Prometheus Node Exporter won't work without it |
| 13 | **Update `deployment.yml` image before Argo CD sync** | GitHub repo | Argo CD will deploy old/wrong image if not updated |

### 🟢 Must Do in Grafana UI (No Commands — All Clicks)

| # | Task |
|---|---|
| 14 | Add Prometheus as a Data Source (`http://<monitoring-ip>:9090`) |
| 15 | Import Jenkins dashboard (ID: `9964`) |
| 16 | Import Node Exporter dashboard (ID: `1860`) |

### ⏳ Steps That Take Time — Don't Close Terminal

| Step | Wait Time |
|---|---|
| EKS Cluster creation | ~20 minutes |
| Node Group creation | ~10–15 minutes |
| Argo CD LoadBalancer URL | ~5 minutes |
| OWASP scan in pipeline | ~30 minutes |
