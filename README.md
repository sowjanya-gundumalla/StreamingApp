# StreamingApp — DevOps Orchestration & Scaling

Stream premium video content, host live watch parties, and manage your catalogue with a modern microservice architecture. This repository contains the full application along with a production-grade CI/CD pipeline, Kubernetes deployment, and monitoring setup.

## Architecture

| Service | Port | Description |
| --- | --- | --- |
| `authService` | 3001 | User authentication, registration, JWT issuance |
| `streamingService` | 3002 | Video catalogue, S3 playback endpoints, public APIs |
| `adminService` | 3003 | Dedicated admin microservice for asset management and uploads |
| `chatService` | 3004 | Websocket + REST chat for live watch parties |
| `frontend` | 3000 | React SPA with revamped UI and integrated chat |
| `mongo` | 27017 | Shared MongoDB instance |

All backend services share common database models and utilities through `backend/common`.

---

## DevOps Stack

| Tool | Purpose |
| --- | --- |
| **GitHub** | Source control, webhook trigger for CI/CD |
| **Docker** | Containerization of all microservices |
| **Amazon ECR** | Private Docker image registry |
| **Jenkins** | CI/CD server on AWS EC2 |
| **Amazon EKS** | Kubernetes cluster for deployment |
| **Helm** | Kubernetes package manager for deployment |
| **Amazon CloudWatch** | Metrics, alarms, and observability |

---

## Step 1: Version Control with Git

### Fork & Clone
```bash
# Clone your forked repository
git clone https://github.com/sowjanya-gundumalla/StreamingApp.git
cd StreamingApp

# Add upstream to sync with original
git remote add upstream https://github.com/UnpredictablePrashant/StreamingApp.git
git fetch upstream
git merge upstream/main
```

---

## Step 2: Docker Containerization

Each microservice has its own Dockerfile. Docker Compose is used for local development.

### Running Locally with Docker Compose
```bash
# Build and start all services
docker-compose up --build

# Access the app at
http://localhost:3000
```

### Services and Ports
| Service | Local Port |
| --- | --- |
| Frontend | 3000 |
| Auth Service | 3001 |
| Streaming Service | 3002 |
| Admin Service | 3003 |
| Chat Service | 3004 |
| MongoDB | 27017 |

---

## Step 3: AWS ECR — Container Registry

### ECR Repositories
| Repository | URI |
| --- | --- |
| streaming-frontend | 503577850130.dkr.ecr.ap-south-1.amazonaws.com/streaming-frontend |
| streaming-auth | 503577850130.dkr.ecr.ap-south-1.amazonaws.com/streaming-auth |
| streaming-service | 503577850130.dkr.ecr.ap-south-1.amazonaws.com/streaming-service |
| streaming-admin | 503577850130.dkr.ecr.ap-south-1.amazonaws.com/streaming-admin |
| streaming-chat | 503577850130.dkr.ecr.ap-south-1.amazonaws.com/streaming-chat |

### Push Images to ECR
```bash
# Authenticate Docker to ECR
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 503577850130.dkr.ecr.ap-south-1.amazonaws.com

# Tag and push (example for frontend)
docker build -t streaming-frontend ./frontend
docker tag streaming-frontend:latest 503577850130.dkr.ecr.ap-south-1.amazonaws.com/streaming-frontend:latest
docker push 503577850130.dkr.ecr.ap-south-1.amazonaws.com/streaming-frontend:latest
```

---

## Step 4: CI/CD with Jenkins on AWS EC2

### Jenkins Setup
- **EC2 Instance:** t3.small, Ubuntu 24.04, ap-south-1
- **Jenkins Version:** 2.555.2
- **Java:** OpenJDK 21
- **Jenkins URL:** http://\<EC2-IP\>:8080

### Jenkins Installation
```bash
# Install Java 21
sudo apt install fontconfig openjdk-21-jre -y

# Add Jenkins repo and install
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo gpg --dearmor -o /usr/share/keyrings/jenkins-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.gpg] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list
sudo apt update && sudo apt install jenkins -y

# Install Docker and add Jenkins to docker group
sudo apt install docker.io -y
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins

# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install
```

### Jenkins Pipeline
The `Jenkinsfile` in the root of this repository defines the CI/CD pipeline with the following stages:

1. **Checkout** — Clones the repository
2. **Login to ECR** — Authenticates Docker with Amazon ECR
3. **Build and Push Frontend** — Builds and pushes frontend image
4. **Build and Push Auth** — Builds and pushes auth service image
5. **Build and Push Streaming** — Builds and pushes streaming service image
6. **Build and Push Admin** — Builds and pushes admin service image
7. **Build and Push Chat** — Builds and pushes chat service image

### GitHub Webhook — Auto Trigger
A GitHub webhook is configured to automatically trigger the Jenkins pipeline on every push to the `main` branch.

```
Payload URL: http://<EC2-IP>:8080/github-webhook/
Content type: application/json
Trigger: Push events
```

---

## Step 5: Kubernetes Deployment with EKS & Helm

### Prerequisites
```bash
# Install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Create EKS Cluster
```bash
eksctl create cluster \
  --name streaming-cluster \
  --region ap-south-1 \
  --nodegroup-name streaming-nodes \
  --node-type t3.small \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 2 \
  --managed
```

### Configure kubectl
```bash
aws eks update-kubeconfig --region ap-south-1 --name streaming-cluster
kubectl get nodes
```

### Deploy with Helm
```bash
# Deploy the application
helm install streaming-app ./helm/streaming-app

# Verify pods are running
kubectl get pods

# Get LoadBalancer URL
kubectl get services
```

### Helm Chart Structure
```
helm/streaming-app/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml   # Deployments for all 5 services
    └── service.yaml      # Services (Frontend: LoadBalancer, Others: ClusterIP)
```

---

## Step 6: Monitoring with Amazon CloudWatch

### Install CloudWatch Observability Addon
```bash
aws eks create-addon \
  --cluster-name streaming-cluster \
  --addon-name amazon-cloudwatch-observability \
  --region ap-south-1
```

### Create CPU Alarm
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "EKS-CPU-High" \
  --alarm-description "EKS CPU utilization high" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --region ap-south-1
```

---

## Step 7: Final Validation

| Check | Status |
| --- | --- |
| GitHub repository forked and maintained | ✅ |
| Docker images built for all 5 microservices | ✅ |
| All 5 ECR repositories created and images pushed | ✅ |
| Jenkins installed and configured on EC2 | ✅ |
| Jenkins pipeline builds and pushes to ECR | ✅ |
| GitHub webhook auto-triggers Jenkins on push | ✅ |
| EKS cluster created with 2 worker nodes | ✅ |
| All 5 pods running on Kubernetes | ✅ |
| Frontend accessible via AWS LoadBalancer | ✅ |
| CloudWatch monitoring configured | ✅ |
| Helm charts committed to GitHub | ✅ |

---

## Environment Configuration

### Auth Service (`backend/authService/.env`)
```ini
PORT=3001
MONGO_URI=mongodb://localhost:27017/streamingapp
JWT_SECRET=changeme
CLIENT_URLS=http://localhost:3000
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=ap-south-1
AWS_S3_BUCKET=
```

### Streaming Service (`backend/streamingService/.env`)
```ini
PORT=3002
MONGO_URI=mongodb://localhost:27017/streamingapp
JWT_SECRET=changeme
CLIENT_URLS=http://localhost:3000
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=ap-south-1
AWS_S3_BUCKET=
AWS_CDN_URL=
STREAMING_PUBLIC_URL=http://localhost:3002
```

### Admin Service (`backend/adminService/.env`)
```ini
PORT=3003
MONGO_URI=mongodb://localhost:27017/streamingapp
JWT_SECRET=changeme
CLIENT_URLS=http://localhost:3000
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=ap-south-1
AWS_S3_BUCKET=
```

### Chat Service (`backend/chatService/.env`)
```ini
PORT=3004
MONGO_URI=mongodb://localhost:27017/streamingapp
JWT_SECRET=changeme
CLIENT_URLS=http://localhost:3000
```

### Frontend (`frontend/.env`)
```ini
REACT_APP_AUTH_API_URL=http://localhost:3001/api
REACT_APP_STREAMING_API_URL=http://localhost:3002/api
REACT_APP_STREAMING_PUBLIC_URL=http://localhost:3002
REACT_APP_ADMIN_API_URL=http://localhost:3003/api/admin
REACT_APP_CHAT_API_URL=http://localhost:3004/api/chat
REACT_APP_CHAT_SOCKET_URL=http://localhost:3004
```

---

## Feature Highlights

- **S3-backed adaptive streaming** with secure signed uploads for admins
- **Dedicated admin microservice** for video ingestion, metadata management, and featured curation
- **Real-time chat** overlay in the player (Socket.IO + persistent message history)
- **Modern React experience** featuring cinematic hero sections, dynamic carousels, and responsive design
- **Role-aware access control** across frontend routes and backend microservices
- **Full CI/CD pipeline** with Jenkins, GitHub webhooks, and Amazon ECR
- **Kubernetes deployment** via Amazon EKS and Helm charts
- **CloudWatch monitoring** with CPU alarms and Container Insights

---

## License

MIT © StreamFlix Team