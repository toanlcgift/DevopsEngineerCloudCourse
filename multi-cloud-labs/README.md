# Multi-Cloud Hands-on Labs

Step-by-step labs for **WSL 2 running Ubuntu 24.04**. Each lab has its own `README.md` with the commands to run, including commands that create or write files.

## Start here

These labs default to local containers and emulators. They do not require AWS, Azure, or Google Cloud accounts and do not create billable cloud resources unless a guide explicitly says otherwise. Emulators do not prove that a real provider's IAM, networking, availability, or managed-service behavior is correct.

### Prepare WSL Ubuntu 24.04

Run these commands in the Ubuntu WSL terminal, not Windows PowerShell:

```bash
echo "Step 1/3: Confirm Ubuntu 24.04 and update packages"
grep -E '^(NAME|VERSION_ID)=' /etc/os-release
sudo apt-get update
sudo apt-get upgrade -y

echo "Step 2/3: Install common lab tools"
sudo apt-get install -y ca-certificates curl git jq make python3 python3-pip \
  python3-venv unzip gnupg lsb-release openssl

echo "Step 3/3: Create a workspace and Python virtual environment"
mkdir -p ~/devops-labs
python3 -m venv ~/devops-labs/.venv
source ~/devops-labs/.venv/bin/activate
python -m pip install --upgrade pip
```

### Docker

If Docker Desktop is installed on Windows, enable **Settings → Resources → WSL Integration → Ubuntu-24.04** and use its Docker CLI from WSL. Verify it with:

```bash
docker version
docker compose version
docker run --rm hello-world
```

Alternatively, install Docker Engine inside WSL. This changes system packages and adds your account to the Docker group, which grants root-equivalent access. Use this only in a trusted WSL distribution. If the `docker` service is not available, enable systemd for WSL using Microsoft's current instructions and restart the distribution.

```bash
echo "Step 1/3: Add Docker's official Ubuntu package repository"
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list >/dev/null

echo "Step 2/3: Install Docker Engine and Compose"
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

echo "Step 3/3: Allow this trusted user to run Docker"
sudo usermod -aG docker "$USER"
```

After adding the group, close and reopen the WSL terminal, then verify:

```bash
docker version
docker compose version
docker run --rm hello-world
```

### Working safely

- Run all commands in Ubuntu WSL Bash. Do not paste Bash commands into PowerShell.
- Keep projects and container bind mounts under the Linux home directory (`~/...`) for better WSL file performance.
- Use only fake credentials with local emulators. Never paste real keys into a command, source file, image, or Git repository.
- The guides bind local web/service ports to `127.0.0.1` where practical. Do not expose lab services to an untrusted network.
- Stop and remove each lab's containers when done. Keep any data you want before removing named volumes.
- Some provider-specific features have no faithful local emulator. Those exercises explain the limitation and keep real-cloud usage optional.

## Labs

| # | Lab | Guide |
|---|---|---|
| 01 | Object Storage Management | [lab-01-object-storage/README.md](lab-01-object-storage/README.md) |
| 02 | Identity and Access Management | [lab-02-identity-access/README.md](lab-02-identity-access/README.md) |
| 03 | Secrets and Configuration Management | [lab-03-secrets-config/README.md](lab-03-secrets-config/README.md) |
| 04 | Serverless Application Development | [lab-04-serverless/README.md](lab-04-serverless/README.md) |
| 05 | NoSQL Database Management | [lab-05-nosql/README.md](lab-05-nosql/README.md) |
| 06 | Messaging and Event-Driven Architecture | [lab-06-messaging/README.md](lab-06-messaging/README.md) |
| 07 | Virtual Machine Deployment | [lab-07-virtual-machines/README.md](lab-07-virtual-machines/README.md) |
| 08 | Containerized Application Deployment | [lab-08-containers/README.md](lab-08-containers/README.md) |
| 09 | Kubernetes Application Deployment | [lab-09-kubernetes/README.md](lab-09-kubernetes/README.md) |
| 10 | Relational Database Deployment | [lab-10-postgresql/README.md](lab-10-postgresql/README.md) |
| 11 | Infrastructure as Code with Terraform | [lab-11-terraform/README.md](lab-11-terraform/README.md) |
| 12 | CI/CD Pipeline with Jenkins | [lab-12-jenkins/README.md](lab-12-jenkins/README.md) |
| 13 | Monitoring and Observability | [lab-13-monitoring/README.md](lab-13-monitoring/README.md) |
| 14 | Configuration Management with Ansible | [lab-14-ansible/README.md](lab-14-ansible/README.md) |
| 15 | Cloud Security and Encryption | [lab-15-security-encryption/README.md](lab-15-security-encryption/README.md) |
| 16 | Event-Driven Data Engineering Pipeline | [lab-16-data-pipeline/README.md](lab-16-data-pipeline/README.md) |
| 17 | AI-Powered RAG Application | [lab-17-rag/README.md](lab-17-rag/README.md) |
| 18 | Multi-Cloud AI Document Management Platform | [lab-18-capstone/README.md](lab-18-capstone/README.md) |

## Suggested order

Complete labs 01–06 first, then 08–10, 11–14, and finally 15–18. Lab 07 uses a local container to demonstrate VM administration concepts; a container is not a virtual machine. Lab 18 is a design-and-integration capstone and refers back to the earlier service labs.
