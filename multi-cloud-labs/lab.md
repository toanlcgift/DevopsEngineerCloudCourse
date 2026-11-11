# Multi-Cloud Hands-on Labs: AWS, Azure & GCP

## Overview

This repository contains practical cloud engineering labs covering three major cloud platforms:

- Amazon Web Services (AWS)
- Microsoft Azure
- Google Cloud Platform (GCP)

The [lab guides](README.md) target WSL 2 with Ubuntu 24.04 and prioritize local execution with Docker, Kubernetes and open-source tools. Cloud emulators have limited service coverage; exercises that cannot be faithfully emulated use local substitutes and explain the differences. No lab creates paid cloud resources by default.

## Lab Environment

| Platform / Tool | Purpose |
|---|---|
| Floci | Local AWS-compatible S3/SQS exercises (limited API coverage) |
| Azurite | Local Azure Blob Storage emulator |
| fake-gcs-server | Local Google Cloud Storage API emulator |
| Docker / Docker Compose | Containerization |
| Minikube / Kind | Local Kubernetes |
| Terraform | Infrastructure as Code |
| Jenkins | CI/CD automation |
| Ansible | Configuration management |
| Prometheus | Metrics collection |
| Grafana | Monitoring dashboards |
| PostgreSQL | Relational database |
| Redis | Caching |
| Ollama | Local LLM inference |
| Qdrant | Vector database |
| Open WebUI | AI interface |
| OpenBao / OPA | Local secrets-management and policy exercises |

---

# Lab List

## Module 1: Cloud Fundamentals and Identity Management

### Lab 01 – Object Storage Management
- Create storage buckets and containers.
- Upload, download and delete objects.
- Configure storage access policies.
- Enable object versioning.
- Compare object storage across cloud platforms.

**Services:**
- AWS: S3
- Azure: Blob Storage
- GCP: Cloud Storage

### Lab 02 – Identity and Access Management
- Create users, groups and service identities.
- Configure roles and permissions.
- Implement least-privilege access.
- Test authorized and unauthorized operations.

**Services:**
- AWS: IAM
- Azure: Entra ID / Azure RBAC
- GCP: IAM / Service Accounts

### Lab 03 – Secrets and Configuration Management
- Store application credentials.
- Retrieve secrets from applications.
- Configure environment-specific settings.
- Implement secure credential management.

**Services:**
- AWS: Secrets Manager
- Azure: Key Vault
- GCP: Secret Manager

---

## Module 2: Serverless and Database Services

### Lab 04 – Serverless Application Development
- Develop Python serverless functions.
- Process HTTP requests.
- Configure environment variables.
- Integrate functions with storage services.

**Services:**
- AWS: Lambda
- Azure: Azure Functions
- GCP: Cloud Functions

### Lab 05 – NoSQL Database Management
- Create a NoSQL database.
- Implement CRUD operations.
- Design partition keys and document structures.
- Connect Python applications to databases.

**Services:**
- AWS: DynamoDB
- Azure: Cosmos DB
- GCP: Firestore

### Lab 06 – Messaging and Event-Driven Architecture
- Create message queues and topics.
- Publish and consume messages.
- Implement asynchronous processing.
- Handle failed messages and retries.
- Build an event-driven order processing system.

**Services:**
- AWS: SQS / SNS
- Azure: Service Bus
- GCP: Pub/Sub

---

## Module 3: Compute and Container Technologies

### Lab 07 – Virtual Machine Deployment
- Create virtual machines.
- Configure networking and security rules.
- Install and configure a web server.
- Deploy a sample web application.

**Services:**
- AWS: EC2
- Azure: Virtual Machines
- GCP: Compute Engine

### Lab 08 – Containerized Application Deployment
- Build Docker images.
- Configure container networking.
- Deploy multi-container applications.
- Configure container environment variables.
- Implement application health checks.

**Services:**
- AWS: ECS / ECR
- Azure: Container Apps / ACR
- GCP: Cloud Run / Artifact Registry

### Lab 09 – Kubernetes Application Deployment
- Deploy a local Kubernetes cluster.
- Deploy applications using Kubernetes manifests.
- Configure Deployments and Services.
- Implement ConfigMaps and Secrets.
- Configure Ingress and persistent storage.
- Perform application scaling and rolling updates.

**Services:**
- AWS: EKS concepts
- Azure: AKS concepts
- GCP: GKE concepts

**Local environment:** Minikube or Kind.

### Lab 10 – Relational Database Deployment
- Deploy PostgreSQL.
- Create databases and tables.
- Configure database users and permissions.
- Connect applications to databases.
- Implement database backup and recovery.

**Services:**
- AWS: RDS
- Azure: Azure Database for PostgreSQL
- GCP: Cloud SQL

---

## Module 4: DevOps, Infrastructure as Code and Monitoring

### Lab 11 – Infrastructure as Code with Terraform
- Install and configure Terraform.
- Define cloud infrastructure using HCL.
- Create reusable modules.
- Manage variables and outputs.
- Understand Terraform state.
- Provision and destroy infrastructure.

**Providers:**
- AWS: AWS Provider
- Azure: AzureRM Provider
- GCP: Google Provider

### Lab 12 – CI/CD Pipeline with Jenkins
- Install Jenkins.
- Create a source code repository.
- Build a Jenkins pipeline.
- Build and test Docker images.
- Deploy applications automatically.
- Integrate Terraform into CI/CD workflows.

**Tools:** Jenkins, Git, Docker, Terraform and cloud CLIs.

### Lab 13 – Monitoring and Observability
- Collect application and infrastructure metrics.
- Centralize application logs.
- Build monitoring dashboards.
- Configure alerting rules.
- Simulate service failures and investigate incidents.

**Services / Tools:**
- AWS: CloudWatch
- Azure: Azure Monitor
- GCP: Cloud Monitoring
- Shared: Prometheus and Grafana

### Lab 14 – Configuration Management with Ansible
- Install and configure Ansible.
- Create inventory files.
- Write reusable playbooks.
- Automate Linux server configuration.
- Deploy and configure web applications.
- Integrate Ansible with Jenkins.

**Tools:** Ansible, Docker and Linux.

---

## Module 5: Cloud Security, Data Engineering and AI

### Lab 15 – Cloud Security and Encryption
- Implement identity-based access control.
- Encrypt application data.
- Manage encryption keys.
- Store application credentials securely.
- Perform security configuration reviews.

**Services:**
- AWS: IAM / KMS
- Azure: Azure RBAC / Key Vault
- GCP: IAM / Cloud KMS

### Lab 16 – Event-Driven Data Engineering Pipeline
- Upload datasets to cloud object storage.
- Trigger event-driven processing.
- Process data using Python.
- Publish processing events to message queues.
- Store processed data in PostgreSQL.
- Build a basic data processing workflow.

**Services:**
- AWS: S3 / Lambda / SQS
- Azure: Blob Storage / Functions / Service Bus
- GCP: Cloud Storage / Cloud Functions / Pub/Sub

### Lab 17 – AI-Powered RAG Application
- Deploy Ollama as a local LLM runtime.
- Deploy Open WebUI.
- Deploy Qdrant as a vector database.
- Build a document ingestion pipeline.
- Generate document embeddings.
- Implement semantic search.
- Build a Retrieval-Augmented Generation (RAG) application.
- Integrate cloud object storage.
- Compare document storage and event processing across cloud platforms.

**Tools:** Ollama, Open WebUI, Qdrant, PostgreSQL, Python and Docker.

---

# Final Project

## Lab 18 – Build a Multi-Cloud AI Document Management Platform

### Project Requirements

Build an AI-powered document management platform using AWS, Azure and GCP service equivalents.

### Architecture Components

| Component | AWS | Azure | GCP |
|---|---|---|---|
| Object Storage | S3 | Blob Storage | Cloud Storage |
| Serverless Processing | Lambda | Functions | Cloud Functions |
| Messaging | SQS | Service Bus | Pub/Sub |
| NoSQL Database | DynamoDB | Cosmos DB | Firestore |
| Container Deployment | ECS | Container Apps | Cloud Run |

### Shared AI Components
- Ollama: LLM inference.
- Qdrant: Vector database.
- Open WebUI: AI chat interface.
- PostgreSQL: Application metadata.
- Python: Document processing and RAG pipeline.

### Final Deliverables
1. Architecture diagrams for all three cloud platforms.
2. Infrastructure as Code configurations.
3. Docker Compose deployment files.
4. Kubernetes deployment manifests.
5. Jenkins CI/CD pipeline.
6. Application source code.
7. Monitoring dashboards.
8. Technical documentation.

### Assessment Criteria
- Cloud architecture understanding.
- Infrastructure provisioning.
- Application deployment.
- Database integration.
- Security implementation.
- CI/CD automation.
- Monitoring and troubleshooting.
- Multi-cloud architecture comparison.

---

## Repository Structure

```text
multi-cloud-labs/
│
├── README.md
├── lab.md
├── lab-01-object-storage/
│   └── README.md
├── lab-02-identity-access/
│   └── README.md
├── lab-03-secrets-config/
│   └── README.md
├── lab-04-serverless/
│   └── README.md
├── lab-05-nosql/
│   └── README.md
├── lab-06-messaging/
│   └── README.md
├── lab-07-virtual-machines/
│   └── README.md
├── lab-08-containers/
│   └── README.md
├── lab-09-kubernetes/
│   └── README.md
├── lab-10-postgresql/
│   └── README.md
├── lab-11-terraform/
│   └── README.md
├── lab-12-jenkins/
│   └── README.md
├── lab-13-monitoring/
│   └── README.md
├── lab-14-ansible/
│   └── README.md
├── lab-15-security-encryption/
│   └── README.md
├── lab-16-data-pipeline/
│   └── README.md
├── lab-17-rag/
│   └── README.md
└── lab-18-capstone/
    └── README.md
```

## Notes

- Prioritize local execution to minimize cloud costs.
- Verify emulator support before implementing each cloud-specific lab.
- Use real Docker-backed services or open-source alternatives when an emulator only provides partial API simulation.
- Use actual cloud accounts only for optional exercises that require production-managed services.
- Keep all credentials and secrets out of source control.

**Goal:** Students should be able to design, deploy, automate, secure and troubleshoot cloud-native applications across AWS, Azure and GCP.