# 🚀 TravellerHub – Cloud-Native DevOps Platform

A production-grade cloud-native travel platform built using the MERN stack and deployed on AWS using modern DevOps, GitOps, Kubernetes, observability, and security best practices.

## 📖 Overview

TravellerHub is based on the Wanderlust TypeScript MERN application and demonstrates a complete end-to-end DevOps workflow:

* Infrastructure on AWS
* Containerization with Docker
* CI/CD using Jenkins
* GitOps with ArgoCD
* Kubernetes on Amazon EKS
* Security Scanning (OWASP Dependency Check, Trivy)
* Code Quality Analysis (SonarQube)
* Monitoring with Prometheus & Grafana
* Redis Caching
* MongoDB Persistence

---

## 🏗️ Architecture

```text
Developer
    │
    ▼
GitHub Repository
    │
    ▼
Jenkins CI Pipeline
 ├── SonarQube Analysis
 ├── OWASP Dependency Check
 ├── Trivy Security Scan
 ├── Docker Build
 └── Docker Push
    │
    ▼
GitOps Repository Update
    │
    ▼
ArgoCD
    │
    ▼
Amazon EKS
 ├── Frontend (React + Vite)
 ├── Backend (Node.js + Express)
 ├── MongoDB
 └── Redis
    │
    ▼
Prometheus + Grafana
```

---

## 🛠️ Tech Stack

### Frontend

* React
* TypeScript
* Vite
* Nginx

### Backend

* Node.js
* Express.js
* TypeScript
* JWT Authentication

### Database & Cache

* MongoDB
* Redis

### DevOps

* Docker
* Jenkins
* SonarQube
* OWASP Dependency Check
* Trivy
* ArgoCD

### Cloud & Infrastructure

* AWS EC2
* Amazon EKS
* IAM
* VPC
* Security Groups
* EBS CSI Driver

### Monitoring

* Prometheus
* Grafana

---

## 📂 Repository Structure

```text
travellerhub/
├── backend/
├── frontend/
├── docker-compose.yml
├── Jenkinsfile
├── kubernetes/
│   ├── namespace.yaml
│   ├── mongodb.yaml
│   ├── redis.yaml
│   ├── backend.yaml
│   └── frontend.yaml
├── GitOps/
│   └── argocd-application.yaml
└── README.md
```

---

## ⚙️ Local Development

### Clone Repository

```bash
git clone https://github.com/sahilsahu246/travellerhub.git
cd travellerhub
```

### Configure Environment

```bash
cp backend/.env.sample backend/.env
cp frontend/.env.sample frontend/.env
```

### Start Application

```bash
docker compose up -d --build
```

### Seed Database

```bash
docker cp backend/data/sample_posts.json mongo:/sample_posts.json

docker compose exec mongo mongoimport \
--db wanderlust \
--collection posts \
--file /sample_posts.json \
--jsonArray
```

Frontend:

```text
http://localhost:5173
```

---

## ☁️ AWS Infrastructure

### Region

```text
ap-south-1 (Mumbai)
```

### Components

* Custom VPC
* Internet Gateway
* Public & Private Subnets
* Route Tables
* Security Groups
* EC2 Instances
* Amazon EKS Cluster
* Managed Node Groups
* IAM Roles

---

## 🚀 CI/CD Pipeline

### Jenkins Stages

1. Source Checkout
2. Dependency Installation
3. SonarQube Analysis
4. OWASP Dependency Check
5. Trivy Security Scan
6. Docker Build
7. Docker Push
8. GitOps Manifest Update
9. Git Push

---

## 🔄 GitOps Workflow

ArgoCD continuously monitors:

```text
kubernetes/
```

Any manifest change automatically triggers deployment to EKS.

---

## 📊 Monitoring Stack

### Prometheus

Metrics collection for:

* Kubernetes
* Nodes
* Pods
* Containers

### Grafana

Dashboards for:

* Cluster Health
* CPU Usage
* Memory Usage
* Pod Metrics
* Node Metrics

---

## 🔐 Security

### Code Security

* SonarQube
* OWASP Dependency Check

### Container Security

* Trivy Image Scanning

### Cloud Security

* IAM Roles
* Security Groups
* Private Networking

---

## 🌍 Application Access

### Frontend

```text
http://<NODE_PUBLIC_IP>:31000
```

### Backend

```text
http://<NODE_PUBLIC_IP>:31100
```

### Jenkins

```text
http://<MASTER_IP>:8080
```

### SonarQube

```text
http://<MASTER_IP>:9000
```

### ArgoCD

```text
https://<NODE_PUBLIC_IP>:<ARGOCD_NODEPORT>
```

### Grafana

```text
http://<NODE_PUBLIC_IP>:<GRAFANA_NODEPORT>
```

---

## 🎯 Key Learning Outcomes

* AWS Networking
* Kubernetes Administration
* GitOps with ArgoCD
* CI/CD Automation
* Container Security
* Observability
* Infrastructure Management
* Cloud-Native Architecture

---

## 👨‍💻 Author

**Sahil Kumar Sahu**

GitHub: https://github.com/sahilsahu246

---

## ⭐ Support

If you found this project useful, consider giving it a star on GitHub.

---

## 🙏 Special Credits

This project is built upon the excellent open-source work of the Wanderlust project created by Krishna Acharyaa.

A huge thank you to:

- GitHub: https://github.com/krishnaacharyaa
- Original Repository: https://github.com/krishnaacharyaa/wanderlust

While this repository extends the original application with a complete Cloud-Native DevOps ecosystem including:

- Docker Containerization
- Jenkins CI/CD
- SonarQube Code Analysis
- OWASP Dependency Check
- Trivy Security Scanning
- Amazon EKS
- Kubernetes Deployments
- ArgoCD GitOps
- Prometheus Monitoring
- Grafana Dashboards
- AWS Infrastructure Setup

The core application functionality and foundation were originally developed by Krishna Acharyaa. This project serves as a DevOps transformation and production-grade deployment implementation of the original Wanderlust application.

Special thanks for creating and open-sourcing such an amazing project for the community. 🚀

<p align="center">
  <a href="https://github.com/krishnaacharyaa/wanderlust">
    <img src="https://img.shields.io/badge/Based%20on-Wanderlust-blue?style=for-the-badge" />
  </a>
  <img src="https://img.shields.io/badge/AWS-EKS-orange?style=for-the-badge" />
  <img src="https://img.shields.io/badge/GitOps-ArgoCD-red?style=for-the-badge" />
  <img src="https://img.shields.io/badge/CI%2FCD-Jenkins-green?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Monitoring-Prometheus%20%26%20Grafana-yellow?style=for-the-badge" />
</p>