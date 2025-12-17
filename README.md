# Amazon Shopping Website DevSecOps Pipelines — Setup Guide

This README collects useful commands and links to install common DevOps, CI/CD, and security tooling on Ubuntu systems. It has been cleaned up, organized, and corrected for clarity. Always review commands for your environment and needs.

> **Note:** Replace all `<VERSION>`, `<your-server-ip>`, `<jenkins-ip>`, `<sonar-ip-address>`, `<ACCOUNT_ID>`, and similar placeholders with your actual values.
---


- [Prerequisites](#prerequisites)
- [System Update & Common Packages](#system-update--common-packages)
- [Java](#java)
- [Jenkins](#jenkins)
- [Docker](#docker)
- [Trivy](#trivy-vulnerability-scanner)
- [Prometheus](#prometheus)
- [Node Exporter](#node-exporter)
- [Grafana](#grafana)
- [Jenkins Plugins to Install](#jenkins-plugins-to-install)
- [Jenkins Credentials to Store](#jenkins-credentials-to-store)
- [Jenkins Tools Configuration](#jenkins-tools-configuration)
- [Jenkins System Configuration](#jenkins-system-configuration)
- kubernetes configuration


---

<img width="1408" height="768" alt="DevSecOps" src="https://github.com/user-attachments/assets/f68f626d-7ed9-49de-a03b-35e675ae4b52" />




## Ports to Enable in Security Group

| Service         | Port  |
|-----------------|-------|
| HTTP            | 80    |
| HTTPS           | 443   |
| SSH             | 22    |
| Jenkins         | 8080  |
| SonarQube       | 9000  |
| Prometheus      | 9090  |
| Node Exporter   | 9100  | 
| Grafana         | 3000  |

---

## Prerequisites

This guide assumes an Ubuntu/Debian-like environment and sudo privileges.

---

## System Update & Common Packages

```bash
sudo apt update
sudo apt upgrade -y

# Common tools
sudo apt install -y bash-completion wget git zip unzip curl jq net-tools build-essential ca-certificates apt-transport-https gnupg fontconfig
```
Reload bash completion if needed:
```bash
source /etc/bash_completion
```

**Install latest Git:**
```bash
sudo add-apt-repository ppa:git-core/ppa
sudo apt update
sudo apt install git -y
```

---

## Java

Install OpenJDK (choose 17 or 21 depending on your needs):

```bash
# OpenJDK 17
sudo apt install -y openjdk-17-jdk

# OR OpenJDK 21
sudo apt install -y openjdk-21-jdk
```
Verify:
```bash
java --version
```

---

## Jenkins

Official docs: https://www.jenkins.io/doc/book/installing/linux/

```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install -y jenkins
sudo systemctl enable --now jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```
Initial admin password:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
Then open: http://your-server-ip:8080

**Note:** Jenkins requires a compatible Java runtime. Check the Jenkins documentation for supported Java versions.

---

## Docker

Official docs: https://docs.docker.com/engine/install/ubuntu/

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add user to docker group (log out / in or newgrp to apply)
sudo usermod -aG docker $USER
newgrp docker
docker ps
```
If Jenkins needs Docker access:
```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```
Check Docker status:
```bash
sudo systemctl status docker
```

---

## Trivy (Vulnerability Scanner)

Docs: https://trivy.dev/v0.65/getting-started/installation/

```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install -y trivy


trivy --version
```

---

## Prometheus

Official downloads: https://prometheus.io/download/

**Generic install steps:**
```bash
# Create a prometheus user
sudo useradd --system --no-create-home --shell /usr/sbin/nologin prometheus

wget -O prometheus.tar.gz "https://github.com/prometheus/prometheus/releases/download/v3.5.0/prometheus-3.5.0.linux-amd64.tar.gz"
tar -xvf prometheus.tar.gz
cd prometheus-*/

sudo mkdir -p /data /etc/prometheus
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml

sudo chown -R prometheus:prometheus /etc/prometheus /data
```

**Systemd service** (`/etc/systemd/system/prometheus.service`):

```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/data \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090

[Install]
WantedBy=multi-user.target
```

**Enable & start:**
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```
Access: http://ip-address:9090

---

## Node Exporter

Docs: https://prometheus.io/docs/guides/node-exporter/

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin node_exporter

wget -O node_exporter.tar.gz "https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-amd64.tar.gz"
tar -xvf node_exporter.tar.gz
sudo mv node_exporter-*/node_exporter /usr/local/bin/
rm -rf node_exporter*
```
**Systemd service:** (`/etc/systemd/system/node_exporter.service`)
```ini
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
ExecStart=/usr/local/bin/node_exporter --collector.logind

[Install]
WantedBy=multi-user.target
```
Enable & start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```

**Prometheus scrape config:**

Add to `/etc/prometheus/prometheus.yml`:
```yaml
  - job_name: "node_exporter"
    static_configs:
      - targets: ["<ip-address>:9100"]

  - job_name: "jenkins"
    metrics_path: /prometheus
    static_configs:
      - targets: ["<jenkins-ip>:8080"]
```
Validate config:
```bash
promtool check config /etc/prometheus/prometheus.yml
sudo systemctl restart prometheus
```

---

## Grafana

Docs: https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/

```bash
sudo apt-get install -y apt-transport-https software-properties-common wget

sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

sudo apt-get update
sudo apt-get install -y grafana

sudo systemctl daemon-reload
sudo systemctl enable --now grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```
Access: http://ip-address:3000

---

Datasource: http://promethues-ip:9090

## Dashboard id 
  - Node_Exporter 1860
Docs: https://grafana.com/grafana/dashboards/1860-node-exporter-full/
  - jenkins       9964
Docs: https://grafana.com/grafana/dashboards/9964-jenkins-performance-and-health-overview/
  - kubernetes    18283
Docs: https://grafana.com/grafana/dashboards/18283-kubernetes-dashboard/



## Jenkins Plugins to Install

- Eclipse Temurin installer Plugin
- NodeJS
- Email Extension Plugin
- OWASP Dependency-Check Plugin
- Pipeline: Stage View Plugin
- SonarQube Scanner for Jenkins
- Prometheus metrics plugin
- Docker API Plugin
- Docker Commons Plugin
- Docker Pipeline
- Docker plugin
- docker-build-step

---
## SonarQube Docker Container Run for Analysis

```bash
docker run -d --name sonarqube \
  -p 9000:9000 \
  -v sonarqube_data:/opt/sonarqube/data \
  -v sonarqube_logs:/opt/sonarqube/logs \
  -v sonarqube_extensions:/opt/sonarqube/extensions \
  sonarqube:lts-community
```

---

## Jenkins Credentials to Store

| Purpose       | ID            | Type          | Notes                               |
|---------------|---------------|---------------|-------------------------------------|
| SonarQube     | sonar-token   | Secret text   | From SonarQube application         |
| Docker Hub    | docker-cred   | Secret text   | From your Docker Hub profile       |

Webhook example:  
`http://<jenkins-ip>:8080/sonarqube-webhook/`

---

## Jenkins Tools Configuration

- JDK
- SonarQube Scanner installations [sonar-scanner]
- Node
- Dependency-Check installations [dp-check]
- Maven installations

- Docker installations

---

## Jenkins System Configuration

**SonarQube servers:**   
- Name: sonar-server  
- URL: http://<sonar-ip-address>:9000  
- Credentials: Add from Jenkins credentials


---
# Now See the configuration pipeline of the jenkins



## Kubernetes Setup (Single Node Cluster)
1️⃣ Disable swap (MANDATORY)
```
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

2️⃣ Install container runtime (containerd)
```
sudo apt update
sudo apt install -y containerd
```
```
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

3️⃣ Install Kubernetes tools
```
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Verify:

```
kubectl version --client
```

#Initialize Kubernetes Cluster
1️⃣ Initialize cluster
```bash 
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

2️⃣ Configure kubectl for ubuntu user

```bash 
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Test:
```bash
kubectl get nodes
````

You’ll see:
```
STATUS: NotReady
```

That’s normal.

3️⃣ Install CNI (Calico – REQUIRED)
```bash

kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

kubectl get nodes
```

✅ Status should become:
    Ready
    
##Deploy Your Amazon App to Kubernetes

Now the real deployment.

1️⃣ Create namespace
```bash
kubectl create namespace amazon
```

2️⃣ Create Deployment YAML

Create file:
```
nano amazon-deployment.yaml
```


Paste this 👇
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: amazon-app
  namespace: amazon
spec:
  replicas: 2
  selector:
    matchLabels:
      app: amazon
  template:
    metadata:
      labels:
        app: amazon
    spec:
      containers:
      - name: amazon
        image: mithiranaws/amazon:latest
        ports:
        - containerPort: 80

```
Apply:

```
kubectl apply -f amazon-deployment.yaml
```

Check:
```
kubectl get pods -n amazon
```


All pods should be Running.

3️⃣ Expose App (Service)

Create service file:

```
nano amazon-service.yaml
```

Paste:

```
apiVersion: v1
kind: Service
metadata:
  name: amazon-service
  namespace: amazon
spec:
  type: NodePort
  selector:
    app: amazon
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30007
```


Apply:

```
kubectl apply -f amazon-service.yaml
```

Check:

```
kubectl get svc -n amazon
```
#Verify Everything

```
kubectl get pods
kubectl get svc
kubectl describe pod amazon-app
```

##Access the Application
Open browser:
http://<ip_address>:30007
