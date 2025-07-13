# Setup
## Create Jenkins, Docker, and SonarQube Instance with Terraform

```bash
cd terraform/Jenkins-SonarQube-VM
aws configure
terraform init
terraform plan
terraform apply -auto-approve
```

## Jenkins, Docker, SonarQube & Trivy Setup Script (`install.sh`)
```bash
#!/bin/bash
sudo apt update -y

# Install Java
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | tee /etc/apt/keyrings/adoptium.asc
echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | tee /etc/apt/sources.list.d/adoptium.list
sudo apt update -y
sudo apt install temurin-17-jdk -y

# Install Jenkins
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install jenkins -y
sudo systemctl start jenkins

# Install Docker and run SonarQube
sudo apt-get install docker.io -y
sudo usermod -aG docker ubuntu
sudo usermod -aG docker jenkins  
newgrp docker
sudo chmod 777 /var/run/docker.sock
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community

# Install Trivy
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y
```

## Create Monitoring Server with Terraform (Prometheus + Grafana)
```bash
cd terraform/Monitoring-Server
terraform init
terraform plan
terraform apply -auto-approve
```

## Prometheus, Node Exporter & Grafana Setup Script (`install.sh`)
```bash
#!/bin/bash
sudo apt update -y

# Install Prometheus
sudo useradd --system --no-create-home --shell /bin/false prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
cd prometheus-2.47.1.linux-amd64/
sudo mkdir -p /data /etc/prometheus
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
sudo chown -R prometheus:prometheus /etc/prometheus/ /data/

sudo tee /etc/systemd/system/prometheus.service > /dev/null <<EOF
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \\
  --config.file=/etc/prometheus/prometheus.yml \\
  --storage.tsdb.path=/data \\
  --web.console.templates=/etc/prometheus/consoles \\
  --web.console.libraries=/etc/prometheus/console_libraries \\
  --web.listen-address=0.0.0.0:9090 \\
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable prometheus
sudo systemctl start prometheus

# Install Node Exporter
sudo useradd --system --no-create-home --shell /bin/false node_exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
sudo tee /etc/systemd/system/node_exporter.service > /dev/null <<EOF
[Unit]
Description=Node Exporter

[Service]
User=node_exporter
ExecStart=/usr/local/bin/node_exporter --collector.logind

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable node_exporter
sudo systemctl start node_exporter

# Install Grafana
sudo apt-get install -y apt-transport-https software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get -y install grafana
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```
## Prometheus Monitoring Jobs
Add Node Exporter Job:
```bash
sudo nano /etc/prometheus/prometheus.yml
```

```yaml
- job_name: 'node_exporter'
  static_configs:
    - targets: ['<instance-ip>:9100']
```

Add Jenkins Job:
```yaml
- job_name: 'jenkins'
  metrics_path: '/prometheus'
  static_configs:
    - targets: ['<jenkins-ip>:8080']
```

Check config and reload:
```bash
promtool check config /etc/prometheus/prometheus.yml
curl -X POST http://localhost:9090/-/reload
```

## Install Kubernetes Tools on Jenkins Server
```bash
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install

# eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

## Create EKS Cluster
```bash
eksctl create cluster --name youtube-cluster \
--region eu-central-1 \
--node-type t2.small \
--nodes 3
kubectl get nodes
```

## Install Prometheus with Helm on EKS
```bash
# Install Helm
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

# Add Repos
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
kubectl create namespace prometheus

# Install Prometheus
helm install stable prometheus-community/kube-prometheus-stack -n prometheus
```

Expose Prometheus:
```bash
kubectl edit svc stable-kube-prometheus-sta-prometheus -n prometheus
# Change type: ClusterIP → LoadBalancer and set port to 9090
kubectl get svc -n prometheus
```

Trigger Jenkins Pipeline:
```bash
git clone https://github.com/hamdhaitam/a-youtube-clone-app.git
cd a-youtube-clone-app
git add .
git commit -m "trigger build"
git push origin main
```

## Cleanup
```bash
# Kubernetes
kubectl delete --all pods -n prometheus
kubectl delete namespace prometheus
kubectl delete deployment.apps/youtube-cluster
kubectl delete service/youtube-service

# EKS Cluster
eksctl delete cluster --region eu-central-1 --name youtube-cluster

# Terraform resources
terraform destroy
```