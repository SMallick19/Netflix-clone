# 🎬 Netflix Clone - DevSecOps Project
[![LinkedIn](https://img.shields.io/badge/Connect%20with%20me%20on-LinkedIn-blue.svg)](https://www.linkedin.com/in/aditya-yadav-14078425a/)
[![GitHub](https://img.shields.io/github/stars/adityayadav7838.svg?style=social)](https://github.com/Adityayadav7838)

![Architecture Diagram](assets/arch-diag.gif)


## 🏗️ 1. Global Architecture Overview
This project establishes a secure Continuous Integration and Continuous Deployment (CI/CD) lifecycle. Source code changes on GitHub pass through deep security gates, container building pipelines, and automated artifact generation before rolling out to an orchestration layer.  

Built to demonstrate **real-world DevSecOps workflows** for CI/CD, cloud automation, security integration, and observability — all in one Netflix-themed application. 🍿  

---

## 🚀 Project Overview

This project simulates a real enterprise-grade setup where a **React-based Netflix Clone** is deployed and managed through a **secure, automated DevOps pipeline**.

### 🌐 Key Features
- **State management** using Terraform Cloud  
- **CI/CD automation** with GitHub Actions and Jenkins  
- **Security Scanning** with Trivy & OWASP Dependency Check  
- **Containerization** with Docker  
- **Kubernetes Deployment** (unmanaged cluster setup)  
- **Monitoring Stack** for Jenkins, Kubernetes, and the app itself  

---

## 🧩 Directory Structure
```bash
.
                   [Developer Commit]
                          │
                          ▼
                     [GitHub Repo]
                          │
                          ▼
                  [Jenkins Pipeline]
     ┌────────────────────┼────────────────────┐
     ▼                    ▼                    ▼
[SonarQube]         [Trivy Scan]       [Docker Build & Push]
(Code Quality)     (Vulnerabilities)     (To DockerHub)
                                               │
                                               ▼
                                      [ArgoCD / Kubeconfig]
                                               │
                                               ▼
                                      [Kubernetes Cluster]
                                    ┌──────────┴──────────┐
                                    ▼                     ▼
                             [Master Node]          [Worker Node]
                                    │                     │
                                    └──────────┬──────────┘
                                               ▼
                                   [Node Exporter Monitoring]
                                               │
                                               ▼
                                      [Prometheus Server]
                                               │
                                               ▼
                                      [Grafana Dashboard]
                                               │
                                               ▼
                                      [Slack Webhook Bot]
                                      
├── Application-Code        # Frontend Netflix Clone app built with React + Vite
│   ├── Dockerfile           # Docker image build instructions
│   ├── package.json         # Dependencies and scripts
│   ├── src/                 # Main source code
│   └── public/              # Static assets
│
├── Jenkins
│   └── Jenkinsfile          # CI/CD pipeline configuration (build → test → deploy)
│
├── Kubernetes
│   ├── deployment.yml       # App deployment manifest
│   └── service.yml          # K8s service exposure
```

## 🛠️ Tech Stack

| Category | Tools / Technologies |
|-----------|----------------------|
| **Infrastructure** |  AWS EC2,  Cloud |
| **CI/CD** | Jenkins, GitHub Actions |
| **Security** | Trivy, SonarQube, OWASP Dependency Check |
| **Containerization** | Docker |
| **Orchestration** | Kubernetes (Unmanaged Cluster) |
| **Monitoring** | Node Exporter, Prometheus, Kube State Metrics |
| **Frontend** | React, Vite, TMDB API |

---

## 💻 2. Infrastructure Layer Configuration (AWS EC2)
The environment relies on four dedicated Ubuntu 22.04 LTS instances configured with a centralized Security Group to maintain internal network access while filtering external public connections.

** Security Group Strategy **
Internal Access: Allow All Traffic where the source is the Security Group ID itself. This ensures that Master, Worker, Monitoring, and Jenkins nodes communicate natively without restrictive perimeter hurdles.

External Access: Narrow down administration ports to your explicit public IP.

|Instance Name       |    Purpose         | Minimum Instance Type           |    Critical Open Inbound Ports         |
|--------------------|--------------------|---------------------------------|----------------------------------------|
|Jenkins-Server	    |Core Orchestrator,  | t2.large (4GB+ RAM recommended) |    8080 (Jenkins Engine)                |
|                    |Builds, Scans       |                                 |                                        |
|SonarQube-Server	|Static Application  | t2.medium (Minimum 2GB+ RAM)	   |    9000 (Sonar Portal)                    |
|                    |Security Testing    |                                 |                                        |
|K8s-Master          |Kubernetes Control  | t3.medium (2 vCPUs minimum)     |    6443 (API Server), 2379-2380 (etcd) |
|K8s-Worker          |Pod Application     | t2.medium                       |    30000-32767 (NodePort App Services) |
|                    |Executions          |                                 |                                        |
|Monitoring-Server   | Observability      |t2.medium9090                    |    (Prometheus), 3000 (Grafana)        |
|                    |Control Node        |                                 |                                        |

---
## 🛠️ 3. Initialization & Tool Installation
Before running the delivery pipelines, every server requires foundational container engines, orchestration packages, or system performance optimizations.

### A. Docker Engine Setup (Jenkins & Sonar Servers)
```bash
sudo apt update && sudo apt install apt-transport-https ca-certificates curl software-properties-common -y
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.di/docker.list > /dev/null
sudo apt update && sudo apt install docker-ce -y
sudo usermod -aG docker $USER && newgrp docker
```
### B. Linux Virtual Memory Adjustments (Required for SonarQube ElasticSearch)
Without modifying kernel parameters, the heavy database engine inside SonarQube will hit thread-capacity thresholds and crash silently.
```bash 
# Apply temporarily
sudo sysctl -w vm.max_map_count=262144

# Persist across node reboots
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```
### C. Spin Up SonarQube Container
```bash 
docker run -d --name sonarqube -p 9000:9000 -p 9092:9092 sonarqube:lts-community
```
----
## ☸️ 4. Kubernetes Cluster Bootstrapping (kubeadm)
Execute these commands to build the container orchestration plane across your Master and Worker topology.

### Step 1: System Level Prerequisites (Both Master & Worker)
```bash 
# Disable Swap (Mandatory for Kubelet Stability)
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Load Essential Kernel Modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Configure Sysctl settings for bridging networking
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```
### Step 2: Install Containerd Container Runtime
```bash 
sudo apt update && sudo apt install containerd -y
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
```
### Step 3: Add Kubernetes Repositories & Install Packages
```bash
sudo apt update && sudo apt install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update && sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
### Step 4: Control Plane Initialization (Master Node Only)
```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16

# Configure regular user cluster credentials
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install Calico Pod Network Add-on for Container Network Interface (CNI)
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml
```
### Step 5: Worker Registration (Worker Node Only)
Execute the specific token output string generated by the control plane init step above:
```bash
sudo kubeadm join <master-internal-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```
---
## 🚀 5. Automated CI/CD Declarative Pipeline
This production file coordinates the building processes on the Jenkins server. It features specific runtime build parameters (TMDB_V3_API_KEY) to feed live API data directly into the Netflix interface compilation layer.
```bash
pipeline {
    agent any
    
    tools {
        jdk 'jdk'
        nodejs 'nodejs'
    }
    
    environment {
        SCANNER_HOME = tool 'sonar-server' 
    }
    
    stages {
        stage('Workspace Cleaning') {
            steps {
                cleanWs()
            }
        }
        
        stage('Checkout from Git') {
            steps {
                git branch: 'master', url: 'https://github.com/adityayadav7838/Netflix-Clone-K8S-End-to-End-Project.git'
            }
        }
        
        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh "${SCANNER_HOME}/bin/sonar-scanner -Dsonar.projectName=Netflix -Dsonar.projectKey=Netflix"
                }
            }
        }
        
        stage("Quality Gate") {
            steps {
                script {
                    timeout(time: 1, unit: 'HOURS') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }
        
        stage('Install Dependencies') {
            steps {
                dir('Application-Code') {
                    sh "npm install"
                }
            }
        }

        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs Application-Code > trivyfs.txt"
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME'),
                    string(credentialsId: 'tmdb-api', variable: 'TMDB_KEY')]) {
                        dir('Application-Code') {
                            sh "echo ${DOCKER_PASSWORD} | docker login -u ${DOCKER_USERNAME} --password-stdin"
                            sh "docker build --build-arg TMDB_V3_API_KEY=${TMDB_KEY} -t aditya9811/netflix:latest ."
                            sh "docker push aditya9811/netflix:latest"
                            sh "docker logout"
                        }
                    }
                 }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    dir('Kubernetes') {
                        withKubeConfig([credentialsId: 'k8s']) {
                            sh 'kubectl apply -f deployment.yml'
                            sh 'kubectl apply -f service.yml'
                            sh 'kubectl get svc'
                            sh 'kubectl get pods'
                        }
                    }
                }
            }
        }
    }
    
    post {
        always {
            emailext (
                attachLog: true,
                subject: "Build ${currentBuild.result}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """Project: ${env.JOB_NAME}
                         Build Number: ${env.BUILD_NUMBER}
                         Status: ${currentBuild.result}
                         URL: ${env.BUILD_URL}""",
                to: 'sarvjanikmemes@gmail.com' 
            )
            cleanWs() 
        }
        
        success {
            slackSend(color: 'good', message: "SUCCESS: Job '${env.JOB_NAME}' [${env.BUILD_NUMBER}] (${env.BUILD_URL})")
        }
        
        failure {
            slackSend(color: 'danger', message: "FAILED: Job '${env.JOB_NAME}' [${env.BUILD_NUMBER}] (${env.BUILD_URL})")
        }
    }
}
```
---
## 📊 6. Observability Stack Configuration
Enterprise infrastructure requires monitoring lines out of bands from the application runtime. This system collects OS data via specialized exporter daemons.

### A. Exporter Deployment (All cluster instances)
```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/

# Set up systemd execution control configuration file
cat <<EOF | sudo tee /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=ubuntu
ExecStart=/usr/local/bin/node_exporter

[Restart]
always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload && sudo systemctl start node_exporter && sudo systemctl enable node_exporter
```
### B. Prometheus Targets Integration (/etc/prometheus/prometheus.yml)
```bash
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'K8s-Master'
    static_configs:
      - targets: ['172.31.10.11:9100']
        labels:
          alias: 'Master-Node'

  - job_name: 'K8s-Worker'
    static_configs:
      - targets: ['172.31.12.34:9100']
        labels:
          alias: 'Worker-Node'
```
---
## 🚨 7. Enterprise Incident Response Routing (Slack Engine)
The alerting engine translates abstract database statistics into clear actionable incident structures sent to Slack channels using optimized custom headers.

### Grafana Notification Layout Settings
Target Engine Type: Custom Contact Point Channel -> Slack Interceptor Webhook.

Title Configuration: 🚨 Netflix Infrastructure Alert 🚨 (This masks out raw internal system tracking IP details like 172.31.12.34:9100 from public preview layouts).
---
### High-Resolution Alert Body Formatter
```bash
{{ if eq .Status "firing" }}🚨ALARM🚨{{ else }}✅OK✅{{ end }}

Alarm Name: {{ .CommonLabels.alertname }}
State: {{ if eq .Status "firing" }}ALARM{{ else }}OK{{ end }}
Region: AWS-US-West (N. California)
Reason: Threshold Crossed: Current value is {{ .Values.A }} which is {{ if eq .Status "firing" }}greater{{ else }}less{{ end }} than the threshold (85.0).
```
### Critical Alert Condition Configurations
```bash
# 1. High CPU Utilization Percentage (Warning >85% | Critical >90% | Sev1 >95%)
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100)

# 2. Memory Consumption Thresholds
100 * (1 - ((node_memory_MemFree_bytes + node_memory_Buffers_bytes + node_memory_Cached_bytes) / node_memory_MemTotal_bytes))

# 3. Storage Mount Available Capacity Limits
100 - ((node_filesystem_avail_bytes{mountpoint="/"} * 100) / node_filesystem_size_bytes{mountpoint="/"})
```
---
## 🔬 8. System Stress Validation Procedures
To verify the automated alert system works properly without risking live system processes, run this system synthetic load injector on the target instance to verify your alerts work correctly:
```bash
# Install the synthetic load injection library packages
sudo apt update && sudo apt install stress -y

# Max out processing queues to trigger the alert loop thresholds
# (Sustained run over 11 minutes clears your 10-minute warning delay periods)
stress --cpu $(nproc) --timeout 660s
```
## 📽️ How to do this Project?

> This project is documented through a **5-Part YouTube Series**, each building upon the previous one.

| Part | Title | Description |
|------|--------|-------------|
| 🧩 **Part 1** | *AWS Setup* | Infrastructure setup |
| ⚙️ **Part 2** | *Jenkins, Docker, SonarQube, Trivy Setup* | Core CI/CD pipeline foundations |
| 🧠 **Part 3** | *SonarQube + Trivy + TMDB + Pipeline Run* | Running secure pipelines |
| ☸️ **Part 4** | *Kubernetes Cluster Setup + Deployment* | Full app deployment in K8s |
| 📊 **Part 5** | *Monitoring Setup* | End-to-end observability |


---
