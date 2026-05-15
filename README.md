# production-mlops-platform

The complete production-style workflow and end-to-end architecture are documented inside:

```bash
production-mlops-platform.md
```

That file explains the full MLOps pipeline step-by-step, including:

* AWS infrastructure
* Terraform provisioning
* Kubernetes deployment
* MLflow integration
* CI/CD with GitHub Actions
* GitOps with Argo CD
* ALB Ingress + Route53
* Dockerized services
* Secure cloud deployment patterns

---

# Why This Project Is Production-Style

Most online tutorials stop at:

* training a model in a notebook
* building a small API
* deploying a single Docker container

This project focuses on how modern ML systems are actually designed and operated in production environments.

The workflow inside `production-mlops-platform.md` demonstrates a realistic end-to-end architecture that combines:

* MLOps
* Kubernetes
* DevOps
* GitOps
* Infrastructure as Code
* Cloud Networking
* CI/CD automation
* Secure AWS deployment patterns

into one complete workflow.

---

# Production Workflow Overview

```text
Raw Data
   ↓
Amazon S3
   ↓
Data Cleaning EC2
   ↓
GPU Training EC2 + MLflow
   ↓
Model Registry
   ↓
FastAPI Backend
   ↓
Docker Images
   ↓
Amazon ECR
   ↓
Amazon EKS
   ↓
ALB Ingress Controller
   ↓
Route53 + HTTPS
   ↓
GitHub Actions CI
   ↓
Argo CD GitOps CD
```

---

# What Makes This Workflow Production-Oriented

## Infrastructure as Code

Infrastructure is provisioned using Terraform:

* VPC
* Public/private subnets
* NAT Gateway
* EKS cluster
* ECR repositories
* IAM resources
* Route53
* ACM certificates

This mirrors real cloud infrastructure provisioning workflows used in production teams.

---

## Kubernetes-Based Deployment

Applications are deployed on Amazon EKS instead of running directly on EC2.

The workflow includes:

* Deployments
* Services
* Ingress
* Load balancing
* Readiness probes
* Liveness probes
* Resource limits

which are all common production Kubernetes practices.

---

## GitOps Continuous Delivery

The deployment process follows a GitOps model using Argo CD.

Workflow:

1. GitHub Actions builds Docker images
2. Images are pushed to Amazon ECR
3. Kubernetes manifests are updated automatically
4. Argo CD detects Git changes
5. EKS synchronizes automatically

This creates a modern production-style CI/CD workflow.

---

## Secure Cloud Architecture

The project intentionally avoids insecure beginner-level patterns.

Security practices included:

* GitHub OIDC authentication
* No hardcoded AWS credentials
* IAM Roles for Service Accounts (IRSA)
* Private Kubernetes worker nodes
* Restricted security groups
* Blocked S3 public access
* HTTPS via ACM certificates

---

## MLflow Integration

The workflow includes:

* experiment tracking
* model metrics
* artifact storage
* model registry

which are important components of real MLOps systems.

---

## Scalable Container Workflow

The application is fully containerized:

* FastAPI backend
* frontend
* Kubernetes deployment
* Docker image versioning
* ECR registry integration

This reflects how scalable ML systems are commonly deployed.

---

## Real Networking Components

Unlike basic demos, the workflow includes:

* VPC networking
* ALB Ingress
* Route53 DNS
* TLS/HTTPS
* private/public subnet separation

which are critical concepts in production cloud systems.

---

# Technologies Used

## Cloud

* AWS
* S3
* EKS
* ECR
* Route53
* ACM
* IAM
* EC2

## DevOps

* Docker
* Kubernetes
* Terraform
* GitHub Actions
* Argo CD
* Helm

## ML / MLOps

* MLflow
* PyTorch
* Scikit-learn
* FastAPI

---

# Future Improvements

The architecture can be extended with:

* Prometheus
* Grafana
* SonarQube
* model drift monitoring
* data drift detection
* KServe
* distributed training
* canary deployments
* blue/green deployments

---

# Educational Purpose

This repository is intended for:

* MLOps learning
* cloud-native ML workflows
* Kubernetes practice
* DevOps portfolio projects
* production architecture understanding

---

# Disclaimer

This repository is for educational and portfolio purposes only.

The medical image classification workflow presented here is not intended for clinical or diagnostic use.

---

# Author

Raouf Khosravi

AI / ML Engineer
MLOps • Kubernetes • AWS • GitOps
