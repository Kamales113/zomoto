
# Zomato App — DevOps Pipeline & Kubernetes Deployment

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Infrastructure Setup](#2-infrastructure-setup)
3. [Tool Installation](#3-tool-installation)
4. [Jenkins Configuration](#4-jenkins-configuration)
5. [Phase 1 — CI/CD Pipeline (Docker Deployment)](#5-phase-1--cicd-pipeline-docker-deployment)
6. [Monitoring Stack (Prometheus & Grafana)](#6-monitoring-stack-prometheus--grafana)
7. [Phase 2 — Kubernetes Deployment via AWS EKS](#7-phase-2--kubernetes-deployment-via-aws-eks)
8. [Argo CD (GitOps) Setup](#8-argo-cd-gitops-setup)
9. [Final Deployment & Verification](#9-final-deployment--verification)
10. [Troubleshooting](#10-troubleshooting)
11. [Cleanup & Teardown](#11-cleanup--teardown)

---

## 1. Project Overview

This project deploys a **Zomato clone** (Node.js web application) using a two-phase DevOps strategy:

- **Phase 1:** CI/CD pipeline via Jenkins, deploying the app to a standalone Docker container for immediate validation.
- **Phase 2:** Production-grade deployment to a managed **AWS EKS** (Kubernetes) cluster using **GitOps principles via Argo CD**, monitored by Prometheus and Grafana.

### Repository Structure

```
/
├── src/
├── Dockerfile
├── Jenkinsfile
├── package.json
├── package-lock.json
└── kubernetes/
    ├── deployment.yml
    ├── node-service.yml
    └── service.yml
```

### Technical Toolset

| Category | Tools |
|---|---|
| **CI/CD & Orchestration** | Jenkins |
| **Source Control** | GitHub |
| **Artifact Management** | Docker Hub |
| **Code Quality** | SonarQube (LTS Community) |
| **Security Suite** | OWASP Dependency-Check, Trivy, Docker Scout |
| **Monitoring Stack** | Prometheus, Grafana, Node Exporter |
| **Infrastructure & GitOps** | AWS EKS, Argo CD, eksctl, kubectl |
| **Notifications** | Email Extension Plugin (SMTP) |

> **Security Note:** Trivy handles filesystem and image vulnerability scanning. Docker Scout is integrated as a modern supplement, providing real-time recommendations and advanced analysis for the pushed image.

---

## 2. Infrastructure Setup

### EC2 Server Specifications

| Specification | Zomato Server (Jenkins/Docker) | Monitoring Server (Prometheus/Grafana) |
|---|---|---|
| **Instance Type** | t2.large | t2.large |
| **Operating System** | Ubuntu 24.04 LTS | Ubuntu 24.04 LTS |
| **Storage** | 29 GB | 29 GB |
| **Name** | `zomato server` | `monitoring server` |

> **Note:** `t2.micro` cannot handle the multiple integrated tools. Use `t2.large` minimum. Storage is set to 29 GB to remain within the AWS Free Tier 30 GB limit.

### Security Group Port Configuration

| Port | Service | Target Server |
|---|---|---|
| **8080** | Jenkins Dashboard | Zomato Server |
| **9000** | SonarQube | Zomato Server |
| **3000** | Zomato App (Phase 1) / Grafana Dashboard | Zomato Server / Monitoring Server |
| **9090** | Prometheus | Monitoring Server |
| **9100** | Node Exporter | Zomato, Monitoring, & EKS Worker Nodes |
| **30001** | Zomato App (Phase 2) — NodePort | EKS Cluster |

> **Note:** Port 3000 is used on both servers but for different services. Manage carefully across separate instances.

### IAM Configuration

- Create an IAM user named `k8s` with **AdministratorAccess**.
- Generate **Access Key** and **Secret Key** for AWS CLI configuration.
- Create an **IAM OIDC Identity Provider** and associate it with the EKS cluster for IAM roles for Kubernetes service accounts.

### AWS CLI Installation & Connection

```bash
sudo apt update && sudo apt install unzip -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

**Verify:** `aws --version` → Expected output: `2.18.74`

**SSH Connection:**
```bash
ssh -i "your-key.pem" ubuntu@<public-ip>
sudo su
```

---

## 3. Tool Installation

Install all components via shell scripts for reproducibility.

### Jenkins & JDK 17 (`jenkins.sh`)

```bash
#!/bin/bash
sudo apt update
sudo apt install openjdk-17-jdk -y
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt update && sudo apt install jenkins -y
```

```bash
chmod +x jenkins.sh && ./jenkins.sh
```

- Port **8080** must be open in the EC2 Security Group.
- Retrieve initial admin password: `cat /var/lib/jenkins/secrets/initialAdminPassword`
- Access Jenkins at: `http://<public-ip>:8080`

### Docker (`docker.sh`)

```bash
#!/bin/bash
sudo apt install docker.io -y
sudo usermod -aG docker ubuntu
newgrp docker
```

```bash
chmod +x docker.sh && ./docker.sh
```

**Verify:** `docker --version` → Expected output: `27`

### Trivy (`trivy.sh`)

```bash
#!/bin/bash
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt update && sudo apt install trivy -y
```

```bash
chmod +x trivy.sh && ./trivy.sh
```

**Verify:** `trivy --version` → Expected output: `0.56.2`

### Docker Scout

> **Critical Prerequisite:** You must `docker login` before using Docker Scout.

```bash
# Step 1: Login to Docker Hub
docker login -u <your-dockerhub-username>

# Step 2: Install Docker Scout
curl -sSfL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh | sh -s --
```

### SonarQube

Run as a Docker container to isolate code analysis dependencies:

```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

- Port **9000** must be open in the EC2 Security Group.
- Access at: `http://<public-ip>:9000`
- Default credentials: `admin` / `admin` (you will be prompted to change the password immediately).

**Generate Token:**
`Administration → Security → Users → Create Token`
- Name: `token`
- Expiration: 30 days

### Installed Tool Versions Summary

| Tool | Version |
|---|---|
| AWS CLI | 2.18.74 |
| Java JDK | 17 |
| Docker | 27 |
| Trivy | 0.56.2 |
| Jenkins | Latest (via script) |
| Node.js | 23.10 |
| SonarQube Scanner | 6.2.1.4610 |
| OWASP Dependency Check | 11.1.0 |

---

## 4. Jenkins Configuration

### Plugin Installation

Go to **Manage Jenkins → Plugins** and install:

| Plugin | Purpose |
|---|---|
| Pipeline Stage View | Stage-by-stage execution visibility |
| Eclipse Temurin Installer | Java version management |
| SonarQube Scanner | Code quality integration |
| Docker, Docker Commons, Docker Pipeline, Docker API, Docker Build Step | Build/push Docker images |
| Email Extension Template | Success/failure email notifications |
| OWASP Dependency-Check | Dependency vulnerability scanning |
| NodeJS Plugin | Build the Node.js application |
| Prometheus Metrics | Expose Jenkins metrics |

### Global Tool Configuration

Go to **Manage Jenkins → Tools**:

> **Important:** Tool names must match the Jenkinsfile environment variables exactly to prevent build failures.

| Tool | Name (exact) | Version |
|---|---|---|
| JDK | `jdk17` | JDK 17 (17.0.8.1+1 from adoptium.net) |
| SonarQube Scanner | `sonar-scanner` | 6.2.1.4610 |
| Node.js | `node23` | 23.10 |
| Dependency-Check | `dp-check` | 11.1.0 (from GitHub) |
| Docker | `docker` | Latest (from docker.com) |

### Credentials Configuration

Go to **Manage Jenkins → Credentials → Global**:

| Credential | Kind | ID | Details |
|---|---|---|---|
| SonarQube Token | Secret text | `sonar-token` | Token generated from SonarQube UI |
| Docker Hub | Username/Password | `docker` | Docker Hub username & password |
| Gmail App Password | Username/Password | `email-cred` | Gmail address + Google App Password |

### System Configuration

Go to **Manage Jenkins → System**:

**SonarQube Server:**
- Name: `sonar-server`
- URL: `http://<zomato-server-ip>:9000`
- Authentication Token: `sonar-token`

**SonarQube Webhook** (configured in SonarQube UI):
`Configuration → Webhooks → Create`
- Name: `Jenkins`
- URL: `http://<jenkins-server-ip>:8080/sonarqube-webhook/`

**Email (Extended E-mail Notification):**
- SMTP Server: `smtp.gmail.com`
- Port: `465`
- Credentials: `email-cred`
- Use SSL: ✅
- Default Triggers: `Always`, `Failure`, `Success`

---

## 5. Phase 1 — CI/CD Pipeline (Docker Deployment)

### Pipeline Stages (12 Stages)

| # | Stage | Description |
|---|---|---|
| 1 | **Clean Workspace** | Runs `cleanWs()` — removes all artifacts from previous builds |
| 2 | **Git Checkout** | Clones the Zomato source code from GitHub (master branch) |
| 3 | **SonarQube Analysis** | Scans code using `sonar-server` tool with project key `zomato` |
| 4 | **Quality Gate** | Blocks the pipeline if SonarQube standards are not met |
| 5 | **Install Dependencies** | Runs `npm install` to install Node.js packages |
| 6 | **OWASP Scan** | Runs `DP-Check`; generates `dependency-check-report.xml` (⚠️ ~30 min wait) |
| 7 | **Trivy FS Scan** | Scans the filesystem; outputs results to `trivy.txt` for email attachment |
| 8 | **Docker Build** | Builds Docker image locally tagged as `zomato:latest` |
| 9 | **Tag & Push** | Tags as `latest` and pushes to `<dockerhub-user>/zomato:latest` |
| 10 | **Docker Scout** | Analyzes the pushed image for vulnerabilities and recommendations |
| 11 | **Deploy to Container** | Runs the app container on port 3000 |
| 12 | **Email Notification** | Sends build status + `trivy.txt` attachment via SMTP |

**Deploy to Container command:**
```bash
docker run -d --name zomato -p 3000:3000 <dockerhub-user>/zomato:latest
```

### Accessing the Application (Phase 1)

Navigate to: `http://<zomato-server-public-ip>:3000`

> Port **3000** must be open in the EC2 Security Group.

---

## 6. Monitoring Stack (Prometheus & Grafana)

Set up on the **Monitoring Server** (second EC2 instance).

### Prometheus Service Configuration

Create `/etc/systemd/system/prometheus.service`:

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

### Prometheus Scrape Configuration (`prometheus.yml`)

Append the following under `scrape_configs`:

```yaml
scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['<monitoring-server-ip>:9100']

  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['<zomato-server-ip>:8080']

  - job_name: 'kubernetes'
    static_configs:
      - targets: ['<k8s-node-ip>:9100']
```

> **Note:** The `metrics_path: '/prometheus'` is required to capture data from the Jenkins Prometheus plugin.

### Service Management

```bash
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus

sudo systemctl enable node_exporter
sudo systemctl start node_exporter

sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

**Verify status:**
```bash
sudo systemctl status prometheus
sudo systemctl status node_exporter
sudo systemctl status grafana-server
```

Expected output: `active (running)`

**Validate Prometheus config:**
```bash
/usr/local/bin/promtool check config /etc/prometheus/prometheus.yml
```
Expected output: `SUCCESS`

### Ports

| Service | Port |
|---|---|
| Prometheus | 9090 |
| Grafana | 3000 |
| Node Exporter | 9100 |

---

## 7. Phase 2 — Kubernetes Deployment via AWS EKS

### EKS Cluster Creation

```bash
eksctl create cluster \
  --name Castro-Cluster \
  --region ap-northeast-1 \
  --zones ap-northeast-1a,ap-northeast-1b
```

> Wait time: approximately **20 minutes**.

### Associate OIDC Provider (Mandatory)

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster Castro-Cluster \
  --approve \
  --region ap-northeast-1
```

### Node Group Creation

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
  --ssh-public-key=<pem-file-name>
```

> Wait time: approximately **10–15 minutes**.

**Node Group Specifications:**

| Setting | Value |
|---|---|
| Cluster Name | Castro-Cluster |
| Region | Tokyo (ap-northeast-1) |
| Availability Zones | ap-northeast-1a, ap-northeast-1b |
| Node Instance Type | t2.medium |
| Min Nodes | 2 |
| Max Nodes | 4 |
| Node Storage | 20 GB |

---

## 8. Argo CD (GitOps) Setup

> **Critical Warning:** The `kubectl patch` command for the LoadBalancer **frequently fails in VS Code terminals**. It must be run in **Windows Command Prompt (Admin)** or **PowerShell**.

### Installation

```bash
kubectl create namespace argocd

kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "LoadBalancer"}}'
```

Wait ~5 minutes for the LoadBalancer URL to be generated.

### Verify

```bash
kubectl get namespace
kubectl get pods -n argocd
```

### Retrieve Argo CD Initial Password

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd -o jsonpath="{.data.password}" | base64 -d
```

> Run this in **Windows Command Prompt / PowerShell** if VS Code terminal throws errors.

---

## 9. Final Deployment & Verification

### Update Kubernetes Manifests

Before syncing with Argo CD, update `kubernetes/deployment.yml` in your forked repository:

```yaml
# Change this line:
image: <original-image>

# To your Docker Hub image:
image: <your-dockerhub-username>/zomato:latest
```

> `service.yml` is pre-configured with **NodePort 30001** — no changes needed.

### Create Argo CD Application

1. Log in to Argo CD UI (via LoadBalancer URL).
2. **Add Repository:** Connect your forked GitHub repo via HTTPS.
3. **Create Application:**
   - **App Name:** `zomato` *(must be all lowercase)*
   - **Path:** `kubernetes`
   - **Destination:** `https://kubernetes.default.svc`
   - **Namespace:** `default`
4. **Sync** the application.

### Verification

| Test | URL | Expected Result |
|---|---|---|
| Phase 1 (Docker) | `http://<zomato-server-ip>:3000` | Zomato UI loads |
| Phase 2 (EKS) | `http://<eks-node-ip>:30001` | Zomato UI loads |

**Docker verification commands:**
```bash
docker images     # Verify images are present
docker ps         # Verify container is running
```

**Kubernetes verification commands:**
```bash
kubectl get namespace
kubectl get pods -n argocd
kubectl get nodes
```

---

## 10. Troubleshooting

### Port 3000 Conflict (Jenkins Re-run)

**Error:** Pipeline fails at "Deploy to Container" — port 3000 already bound.

**Fix:**
```bash
docker stop zomato
docker rm zomato
```
Then re-run the pipeline.

### OWASP Unstable Build

**Error:** Pipeline flagged as "unstable" (yellow) during OWASP scan.

**Cause:** High-severity vulnerabilities found, or timeout during the scan (~30 min).

**Fix:** The pipeline is configured to continue regardless. Optionally, adjust the Jenkinsfile to ignore the unstable status or remediate the vulnerable packages.

### Prometheus Node Down (0/1 Up)

**Error:** EKS Node Exporter targets show `0/1 up` (red) in Prometheus.

**Cause:** Port 9100 is not open on the EKS Worker Node Security Group.

**Fix:** In the AWS EC2 Console, navigate to the **Node Group's Security Group** and add an inbound rule for port `9100`.

### Argo CD App Creation Failure

**Error:** `application is invalid` / metadata error.

**Cause:** Uppercase letters in the app name.

**Fix:** Use all lowercase for the app name: `zomato`.

### kubectl / Argo CD Commands Failing in VS Code

**Error:** `kubectl patch` or Argo CD password retrieval throws bad request errors.

**Fix:** Open **Windows Command Prompt** or **PowerShell** as Administrator and run the commands there.

---

## 11. Cleanup & Teardown

> ⚠️ Follow this sequence carefully to avoid unexpected AWS charges.

**Step 1 — Delete Node Group:**
```bash
eksctl delete nodegroup \
  --cluster=Castro-Cluster \
  --name=Castro-demo-NG
```

**Step 2 — Delete Cluster:**
```bash
eksctl delete cluster --name=Castro-Cluster
```

**Step 3 — Manual Cleanup:**
Navigate to the **AWS CloudFormation Console** and delete any remaining stacks associated with the EKS cluster.

---

## Appendix: Complete Step-by-Step Workflow Summary

| Step | Action | Wait Time |
|---|---|---|
| 1 | SSH into Zomato Server, install AWS CLI, Jenkins, Docker, Trivy | — |
| 2 | Docker login, run SonarQube container, install Docker Scout | — |
| 3 | Configure Jenkins: plugins, tools, credentials, system settings | — |
| 4 | Create & run Jenkins Pipeline job | ~30 min (OWASP scan) |
| 5 | Set up Monitoring Server: Prometheus, Node Exporter, Grafana | — |
| 6 | Create EKS Cluster (`eksctl create cluster`) | ~20 min |
| 7 | Associate OIDC Provider, create Node Group | ~10–15 min |
| 8 | Install Argo CD, expose via LoadBalancer | ~5 min |
| 9 | Update `deployment.yml`, create Argo CD app, sync | — |
| 10 | Verify app at `http://<node-ip>:30001` | — |
