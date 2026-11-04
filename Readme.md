# DevOps & Cloud Engineering: Hands-on Command-Line Workbook

This workbook turns the topics in `Course.md` into labs you can run mainly from a terminal on an Ubuntu server. Cloud labs use local simulators for **AWS (LocalStack)**, **Google Cloud Storage (fake-gcs-server)**, and **Azure Blob Storage (Azurite)**; Kubernetes exercises use **Minikube**. No real cloud account is required.

Commands are written for Bash on Ubuntu. Run them on a disposable lab host or VM with internet access. Do not expose lab services to an untrusted network, and do not put real credentials or secrets in the exercises.

Each multi-command lab prints progress messages with `echo` so that pasting a block into a terminal shows which step is running. If a command fails, the last printed step marker narrows down where to investigate.

## Topics

- Ubuntu host preparation
- DevOps workflow, Git, and cloud service models
- Git collaboration and Jenkins CI
- Docker, Compose, and Kubernetes with Minikube
- Infrastructure as Code and monitoring
- AWS, Google Cloud, and Azure simulator labs
- DevSecOps and capstone deployment

## Workbook files

- [Ubuntu host setup](labs/SETUP.md)
- [DevOps workflow, Git, and cloud concepts](labs/DEVOPS_GIT_CLOUD.md)
- [Git collaboration and Jenkins CI](labs/GIT_JENKINS_CI.md)
- [Docker monitoring, Kubernetes with Minikube, Jenkins, and Node.js](labs/DOCKER_KUBERNETES.md)
- [Terraform, Ansible, and monitoring](labs/IAC_MONITORING.md)
- [AWS, GCP, Azure simulators, and DevSecOps capstone](labs/CLOUD_DEVSECOPS.md)
- [Completion checklist](labs/COMPLETION_CHECKLIST.md)

Start with the host setup, then work through the topics. The lab files retain the step-by-step `echo` progress messages and troubleshooting guidance.
