# 🎬 Book My Show Clone — DevOps Project
### by **InfraCorps**

---

> A complete end-to-end DevOps pipeline covering Docker deployment, Kubernetes (EKS) orchestration, CI/CD with Jenkins, security scanning, and full-stack monitoring with Prometheus & Grafana.

---

## 📑 Table of Contents

- [Part I: Docker Deployment](#-part-i-docker-deployment)
  - [Step 1: Basic Setup](#step-1-basic-setup)
  - [Step 2: Tools Installation](#step-2-tools-installation)
  - [Step 3: Access Jenkins Dashboard](#step-3-access-jenkins-dashboard)
  - [Step 4: Email Integration](#step-4-email-integration)
  - [Step 5: Create Pipeline Job](#step-5-create-pipeline-job)
- [Part II: K8S Deployment & Monitoring](#-part-ii-k8s-deployment--monitoring)
  - [Step 6: Monitoring the Application](#step-6-monitoring-the-application)

---

# 🐳 PART I: Docker Deployment

---

## Step 1: Basic Setup

### 1.1 Launch VM

Launch **1 EC2 VM** with the following configuration:

| Setting | Value |
|---|---|
| **OS** | Ubuntu 24.04 |
| **Instance Type** | t2.large |
| **Storage** | 28 GB |
| **Name** | `BMS-Server` |

#### 🔐 Security Group — Open the Following Ports

| Type | Protocol | Port Range | Purpose |
|---|---|---|---|
| SMTP | TCP | 25 | Sending emails between mail servers |
| Custom TCP | TCP | 3000–10000 | Node.js, Grafana, Jenkins, custom apps |
| HTTP | TCP | 80 | Unencrypted web traffic (Apache, Nginx) |
| HTTPS | TCP | 443 | Secure web traffic via SSL/TLS |
| SSH | TCP | 22 | Secure Shell remote access |
| Custom TCP | TCP | 6443 | Kubernetes API server |
| SMTPS | TCP | 465 | Secure email via SMTP over SSL/TLS |
| Custom TCP | TCP | 30000–32767 | Kubernetes NodePort service range |

---

### 1.2 Creation of EKS Cluster

#### 1.2.1 Create IAM User

> ⚠️ **Important:** It is not recommended to create an EKS Cluster using the Root Account. Always use a dedicated IAM user.

#### 1.2.2 Attach Policies to the IAM User

Attach the following **managed policies**:

- `AmazonEC2FullAccess`
- `AmazonEKS_CNI_Policy`
- `AmazonEKSClusterPolicy`
- `AmazonEKSWorkerNodePolicy`
- `AWSCloudFormationFullAccess`
- `IAMFullAccess`

Also attach the following **inline policy**:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "eks:*",
            "Resource": "*"
        }
    ]
}
```

#### 1.2.3 Create Access Keys

Create **Access Keys** for the IAM user. This completes the IAM setup with appropriate permissions to create the EKS Cluster.

---

### 1.3 Creation of EKS Cluster

Connect to the `BMS-Server` VM and run:

```bash
sudo apt update
```

#### 1.3.1 Install AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install
aws configure
```

#### 1.3.2 Install KubeCTL

```bash
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client
```

#### 1.3.3 Install EKSCTL

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

#### 1.3.4 Create EKS Cluster

Execute the following commands as **separate sets**:

**(a) Create the EKS Cluster:**

```bash
eksctl create cluster --name=kartik-eks \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b \
                      --version=1.30 \
                      --without-nodegroup
```

> ⏳ This will take **5–10 minutes**. Verify the cluster in the EKS Console.

**(b) Associate IAM OIDC Provider:**

```bash
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster kartik-eks \
    --approve
```

> 💡 This is crucial for enabling **IAM Roles for Service Accounts (IRSA)**. Amazon EKS uses OpenID Connect (OIDC) to authenticate Kubernetes service accounts with IAM roles. Without this, Pods would require node-level IAM roles (granting permissions to ALL Pods on a node).

**(c) Create Node Group:**

> 📝 Replace `<PEM FILE NAME>` with your PEM key name (without the `.pem` extension).

```bash
eksctl create nodegroup --cluster=kartik-eks \
                       --region=us-east-1 \
                       --name=node2 \
                       --node-type=t3.medium \
                       --nodes=3 \
                       --nodes-min=2 \
                       --nodes-max=4 \
                       --node-volume-size=20 \
                       --ssh-access \
                       --ssh-public-key=kartik \
                       --managed \
                       --asg-access \
                       --external-dns-access \
                       --full-ecr-access \
                       --appmesh-access \
                       --alb-ingress-access
```

> ⏳ This will take **5–10 minutes**.

**(d) Open All Traffic in EKS Security Group:**

For internal communication between the control plane and worker nodes, open **All Traffic** in the security group of the EKS Cluster.

---

## Step 2: Tools Installation

### 2.1.1 Install Jenkins

Connect to the Jenkins Server and create the script:

```bash
vi Jenkins.sh
```

Paste the following content:

```bash
#!/bin/bash

# Install OpenJDK 17 JRE Headless
sudo apt install openjdk-17-jre-headless -y

# Download Jenkins GPG key
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

# Add Jenkins repository to package manager sources
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# Update package manager repositories
sudo apt-get update

# Install Jenkins
sudo apt-get install jenkins -y
```

Save and run:

```bash
# esc → :wq
sudo chmod +x jenkins.sh
./jenkins.sh
```

> 🔓 Open **Port 8080** in the Jenkins server Security Group, then access and set up Jenkins.

---

### 2.1.2 Install Docker

```bash
vi docker.sh
```

Paste the following content:

```bash
#!/bin/bash

# Update package manager repositories
sudo apt-get update

# Install necessary dependencies
sudo apt-get install -y ca-certificates curl

# Create directory for Docker GPG key
sudo install -m 0755 -d /etc/apt/keyrings

# Download Docker's GPG key
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc

# Ensure proper permissions for the key
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add Docker repository to Apt sources
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update package manager repositories
sudo apt-get update

sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Save and run:

```bash
# esc → :wq
sudo chmod +x docker.sh
./docker.sh
docker --version
```

**Verify Docker:**

```bash
docker pull hello-world
# If permission error:
sudo chmod 666 /var/run/docker.sock
docker pull hello-world
```

**Login to DockerHub:**

```bash
docker login -u <DockerHubUserName>
# Enter your DockerHub password when prompted
```

---

### 2.1.3 Install Trivy

```bash
vi trivy.sh
```

Paste the following content:

```bash
#!/bin/bash
sudo apt-get install wget apt-transport-https gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```

Save and run:

```bash
# esc → :wq
sudo chmod +x trivy.sh
./trivy.sh
trivy --version
```

---

### 2.2 SonarQube Setup

Connect to the Jenkins Server and run:

```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
docker images
docker ps
```

> 🔓 Open **Port 9000** in the security group.  
> Access SonarQube at `http://<ip>:9000`  
> **Default credentials:** `admin` / `admin` → Set a new password on first login.

---

## Step 3: Access Jenkins Dashboard

### 3.1 Plugin Installation

Install the following plugins in Jenkins:

| Category | Plugins |
|---|---|
| **Java** | Eclipse Temurin Installer |
| **Code Quality** | SonarQube Scanner, OWASP Dependency Check |
| **Node** | NodeJS |
| **Docker** | Docker, Docker Commons, Docker Pipeline, Docker API, docker-build-step |
| **Kubernetes** | Kubernetes, Kubernetes CLI, Kubernetes Client API, Kubernetes Credentials |
| **Pipeline** | Pipeline Stage View, Config File Provider |
| **Monitoring** | Prometheus Metrics |
| **Notifications** | Email Extension Template |

### 3.2 SonarQube Token Creation

Configure the SonarQube server and use the token:

```
squ_69eb05b41575c699579c6ced901eaafae66d63a2
```

### 3.3 Creation of Credentials

Create the required credentials in Jenkins (Docker, SonarQube, email, Kubernetes).

### 3.4 Tools Configuration

Configure the installed tools (JDK 17, Node.js, SonarQube Scanner, Docker) under **Manage Jenkins → Tools**.

### 3.5 System Configuration in Jenkins

Configure SonarQube server, email notifications, and other system settings under **Manage Jenkins → System**.

---

## Step 4: Email Integration

### Configure Gmail App Password

1. Go to **Gmail** → Click on your profile icon → **Your Google Account**
2. Search for **"App Passwords"** in the search box
3. Enter your Gmail password → **Next**
4. App name: `jenkins` → **Create**
5. Copy the generated password (e.g., `fxssvxsvfbartxnt`) — **remove all spaces**

### Add Email Credentials in Jenkins

**Manage Jenkins → Security → Credentials → Global → Add Credentials:**

| Field | Value |
|---|---|
| Kind | Username with Password |
| Scope | Global |
| Username | `<Your Gmail ID>` |
| Password | `<Paste the App Password Token>` |
| ID | `email-creds` |
| Description | `email-creds` |

### Configure Extended Email Notification

**Manage Jenkins → System → Extended Email Notification:**

- **SMTP Server:** `smtp.gmail.com`
- **SMTP Port:** `465`
- **Credentials:** Select `email-creds`
- ✅ Use SSL
- ✅ Use OAuth 2.0
- **Default Content Type:** HTML

### Configure Email Notification

**Manage Jenkins → System → Email Notification:**

- **SMTP Server:** `smtp.gmail.com`
- ✅ Use SMTP Authentication
- **Username:** `<Your Email ID>`
- **Password:** `<App Password Token>`
- ✅ Use SSL
- **SMTP Port:** `465`
- **Reply-to Email:** `<Your Email>`
- **Charset:** `UTF-8`
- ✅ Test configuration by sending test email → Provide recipient email → **Test Configuration**
- Confirm you receive the test email

### Configure Default Triggers

Under **Default Triggers** (use Ctrl+F to find):
- ✅ Always
- ✅ Failure-Any
- ✅ Success

→ **Apply → Save**

### Install NPM

```bash
apt install npm
```

---

## Step 5: Create Pipeline Job

> ⚠️ **Before pasting the pipeline script, make these changes:**
> 1. In `Tag and Push to DockerHub` stage — replace with **your DockerHub username**
> 2. In `Deploy to Container` stage — replace with **your DockerHub username**
> 3. In the `post` actions — replace with **your configured email ID**

---

### 🔧 Pipeline Script — Docker Deployment (Without K8S)

```groovy
pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node23'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
               git branch: 'main', url: 'https://github.com/kartikhegadi/BookMyShow.git'
                sh 'ls -la'  // Verify files after checkout
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh ''' 
                    $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=BMS \
                    -Dsonar.projectKey=BMS 
                    '''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh '''
                cd bookmyshow-app
                ls -la  # Verify package.json exists
                if [ -f package.json ]; then
                    rm -rf node_modules package-lock.json  # Remove old dependencies
                    npm install  # Install fresh dependencies
                else
                    echo "Error: package.json not found in bookmyshow-app!"
                    exit 1
                fi
                '''
            }
        }
        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }
        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh ''' 
                        echo "Building Docker image..."
                        docker build --no-cache -t kartik404/bms:latest -f bookmyshow-app/Dockerfile bookmyshow-app

                        echo "Pushing Docker image to registry..."
                        docker push kartik404/bms:latest
                        '''
                    }
                }
            }
        }
        stage('Deploy to Container') {
            steps {
                sh ''' 
                echo "Stopping and removing old container..."
                docker stop bms || true
                docker rm bms || true

                echo "Running new container on port 3000..."
                docker run -d --restart=always --name bms -p 3000:3000 kartik404/bms:latest

                echo "Checking running containers..."
                docker ps -a

                echo "Fetching logs..."
                sleep 5  # Give time for the app to start
                docker logs bms
                '''
            }
        }
    }
    post {
        always {
            emailext attachLog: true,
                subject: "'${currentBuild.result}'",
                body: "Project: ${env.JOB_NAME}<br/>" +
                      "Build Number: ${env.BUILD_NUMBER}<br/>" +
                      "URL: ${env.BUILD_URL}<br/>",
                to: 'prajwal.ccs.aws@gmail.com',
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}
```

> ✅ Access the BMS App using the **Public IP of BMS-Server** on port `3000`

---

# ☸️ PART II: K8S Deployment & Monitoring

---

## Pre-requisites Before K8S Stage Pipeline

### Identify the Jenkins User

```bash
ps aux | grep jenkins
```

Example output:
```
jenkins   1234  0.0  0.1 123456 7890 ?  Ssl  12:34   0:00 /usr/bin/java -jar /usr/share/jenkins/jenkins.war
```

### Switch to Jenkins User and Configure AWS

```bash
sudo -su jenkins
pwd       # → /home/ubuntu
whoami    # → jenkins
```

Configure AWS credentials:

```bash
aws configure
# Enter your Access Key and Secret Access Key
# This creates credentials at: /var/lib/jenkins/.aws/credentials
```

Verify credentials:

```bash
aws sts get-caller-identity
```

Expected output:
```json
{
    "UserId": "EXAMPLEUSERID",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/example-user"
}
```

### Restart Jenkins and Update Kubeconfig

```bash
exit  # Come out of Jenkins user
sudo systemctl restart jenkins

# Switch back to Jenkins user
sudo -su jenkins
aws eks update-kubeconfig --name kartik-eks --region us-east-1
```

---

### 🔧 Pipeline Script — K8S Deployment (With K8S Stage)

```groovy
pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node23'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_IMAGE = 'kartik404/bms:latest'
        EKS_CLUSTER_NAME = 'kartik-eks'
        AWS_REGION = 'ap-northeast-1'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/kartikhegadi/BookMyShow.git'
                sh 'ls -la'  // Verify files after checkout
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh ''' 
                    $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=BMS \
                        -Dsonar.projectKey=BMS
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                cd bookmyshow-app
                ls -la  # Verify package.json exists
                if [ -f package.json ]; then
                    rm -rf node_modules package-lock.json  # Remove old dependencies
                    npm install  # Install fresh dependencies
                else
                    echo "Error: package.json not found in bookmyshow-app!"
                    exit 1
                fi
                '''
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh ''' 
                        echo "Building Docker image..."
                        docker build --no-cache -t $DOCKER_IMAGE -f bookmyshow-app/Dockerfile bookmyshow-app

                        echo "Pushing Docker image to Docker Hub..."
                        docker push $DOCKER_IMAGE
                        '''
                    }
                }
            }
        }

        stage('Deploy to EKS Cluster') {
            steps {
                script {
                    sh '''
                    echo "Verifying AWS credentials..."
                    aws sts get-caller-identity

                    echo "Configuring kubectl for EKS cluster..."
                    aws eks update-kubeconfig --name $EKS_CLUSTER_NAME --region $AWS_REGION

                    echo "Verifying kubeconfig..."
                    kubectl config view

                    echo "Deploying application to EKS..."
                    kubectl apply -f deployment.yml
                    kubectl apply -f service.yml

                    echo "Verifying deployment..."
                    kubectl get pods
                    kubectl get svc
                    '''
                }
            }
        }
    }

    post {
        always {
            emailext attachLog: true,
                subject: "'${currentBuild.result}'",
                body: "Project: ${env.JOB_NAME}<br/>" +
                      "Build Number: ${env.BUILD_NUMBER}<br/>" +
                      "URL: ${env.BUILD_URL}<br/>",
                to: 'prajwal.ccs.aws@gmail.com',
                attachmentsPattern: 'trivyfs.txt'
        }
    }
}
```

---

## Step 6: Monitoring the Application

**Launch a new VM:**

| Setting | Value |
|---|---|
| **OS** | Ubuntu 22.04 |
| **Instance Type** | t2.medium |
| **Name** | `Monitoring Server` |

---

### 6.1 Connect to the Monitoring Server & Setup Prometheus User

```bash
sudo apt update

sudo useradd \
    --system \
    --no-create-home \
    --shell /bin/false prometheus
```

> **Explanation of flags:**
> - `--system` — Creates a system account
> - `--no-create-home` — No home directory needed for system accounts
> - `--shell /bin/false` — Prevents login as the Prometheus user

---

### 6.2 Download and Install Prometheus

```bash
sudo wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
sudo mkdir -p /data /etc/prometheus
cd prometheus-2.47.1.linux-amd64/
```

Move binaries and configuration:

```bash
# Move Prometheus binary and promtool
sudo mv prometheus promtool /usr/local/bin/

# Move console libraries to the config directory
sudo mv consoles/ console_libraries/ /etc/prometheus/

# Move the main Prometheus configuration file
sudo mv prometheus.yml /etc/prometheus/prometheus.yml

# Set correct ownership
sudo chown -R prometheus:prometheus /etc/prometheus/ /data/
```

Clean up:

```bash
cd
rm -rf prometheus-2.47.1.linux-amd64.tar.gz
```

Verify installation:

```bash
prometheus --version
# Expected: version 2.47.1

prometheus --help
```

---

### Create Prometheus Systemd Service

```bash
sudo vi /etc/systemd/system/prometheus.service
```

Paste the following content:

```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target
StartLimitIntervalSec=500
StartLimitBurst=5

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
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
```

Enable and start Prometheus:

```bash
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```

> 🔓 Open **Port 9090** for the Monitoring Server VM.  
> Access Prometheus at: `http://<public-ip>:9090`  
> ⚠️ If it doesn't load, use `http://` (not `https://`)

**Verify in browser:** Click **Status → Targets** — You should see `Prometheus (1/1 up)`, scraping itself every 15 seconds.

---

### 6.2 (cont.) Install Node Exporter

```bash
# Create system user
sudo useradd --system --no-create-home --shell /bin/false node_exporter

# Download Node Exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz

# Extract, move binary, and clean up
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
rm -rf node_exporter*

# Verify
node_exporter --version
```

Create systemd service:

```bash
sudo vi /etc/systemd/system/node_exporter.service
```

Paste the following content:

```ini
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target
StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/node_exporter --collector.logind

[Install]
WantedBy=multi-user.target
```

Enable and start Node Exporter:

```bash
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
# You should see "active (running)" in green
# Press Ctrl+C to exit
```

---

### 6.3 Configure Prometheus Plugin Integration

Navigate to the Prometheus config file:

```bash
cd /etc/prometheus/
ls -l
sudo vi prometheus.yml
```

Append the following jobs **at the end of the file**:

```yaml
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['<MonitoringVMip>:9100']

  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['<your-jenkins-ip>:<your-jenkins-port>']
```

> ⚠️ Replace `<MonitoringVMip>`, `<your-jenkins-ip>`, and `<your-jenkins-port>` with the appropriate values.  
> **Note:** Keep port `9100` for Node Exporter — do NOT change it to `9090`.

Validate and reload:

```bash
# Validate the configuration
promtool check config /etc/prometheus/prometheus.yml
# Expected: SUCCESS

# Reload Prometheus without restarting
curl -X POST http://localhost:9090/-/reload
```

Access Prometheus targets:

```
http://<your-prometheus-ip>:9090/targets
```

> 🔓 Open **Port 9100** for Monitoring VM if Node Exporter shows `(0/1)` in red.

You should now see all three targets up:
- ✅ `jenkins (1/1 up)`
- ✅ `node_exporter (1/1 up)`
- ✅ `prometheus (1/1 up)`

> Click **Show More** next to `jenkins` to view the scraped metrics link.

---

### 6.4 Install Grafana

Navigate to home directory:

```bash
cd  # Now in ~ path
```

**Step 1 — Install Dependencies:**

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https software-properties-common
```

**Step 2 — Add the GPG Key:**

```bash
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
# Expected output: OK
```

**Step 3 — Add Grafana Repository:**

```bash
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
```

**Step 4 — Update and Install Grafana:**

```bash
sudo apt-get update
sudo apt-get -y install grafana
```

**Step 5 — Enable and Start Grafana:**

```bash
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

**Step 6 — Check Grafana Status:**

```bash
sudo systemctl status grafana-server
# Expected: "Active (running)" in green
# Press Ctrl+C to exit
```

**Step 7 — Access Grafana Web Interface:**

> 🔓 Open **Port 3000** for the Monitoring Server VM.

```
http://<monitoring-server-ip>:3000
```

- **Default credentials:** `admin` / `admin`
- Click **"Skip Now"** or set a new password

---

### 6.5 Add Data Sources in Grafana

Add Prometheus as a data source in Grafana:

1. Go to **Connections → Data Sources → Add data source**
2. Select **Prometheus**
3. Set the URL to `http://localhost:9090`
4. Click **Save & Test**

---

### 6.6 Add Dashboards in Grafana

Import the following community dashboards:

| Dashboard | URL |
|---|---|
| **Node Exporter Full** | [Dashboard #1860](https://grafana.com/grafana/dashboards/1860-node-exporter-full/) |
| **Jenkins Performance & Health Overview** | [Dashboard #9964](https://grafana.com/grafana/dashboards/9964-jenkins-performance-and-health-overview/) |

To import: **Dashboards → New → Import → Enter Dashboard ID → Load**

Click on **Dashboards** in the left pane to view both dashboards.

---

## 🧹 Cleanup

> ⚠️ Once you are done with the project, make sure to **delete all the resources** created to avoid unnecessary AWS costs.

- Delete EKS Cluster and Node Groups
- Terminate all EC2 Instances (BMS-Server, Monitoring Server)
- Remove IAM Users and Access Keys
- Delete Security Groups and related networking resources

---

*Project by **InfraCorps** | DevOps End-to-End Pipeline*
