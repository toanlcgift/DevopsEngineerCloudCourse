# Ubuntu Lab Host Setup

Use Ubuntu Server 22.04 or 24.04. A VM with 4 vCPUs, 8 GB RAM, and 30 GB free disk is a reasonable starting point; Minikube and Jenkins together may need more. Log in as a regular user with `sudo`.

Update the host and install common command-line tools:

```bash
echo "Step 1/2: Update apt and install common tools"
sudo apt-get update
sudo apt-get install -y ca-certificates curl git jq make wget python3 python3-pip \
  python3-venv unzip gnupg lsb-release openssl
echo "Step 2/2: Create the lab workspace"
mkdir -p ~/devops-labs
cd ~/devops-labs
```

Install Docker Engine from Docker's official Ubuntu repository, then allow your user to run Docker. Group membership is effectively root access; only do this on a trusted lab host. Log out and back in after the `usermod` command, then verify with `docker run --rm hello-world`.

```bash
echo "Step 1/3: Add Docker's package signing key and repository"
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list >/dev/null
echo "Step 2/3: Install Docker Engine and Compose"
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
echo "Step 3/3: Grant this trusted lab user Docker access"
sudo usermod -aG docker "$USER"
```

Install only the tools needed for the labs you plan to run. Each tool has its own section and copyable command block; you do not need to run the whole list. The cloud CLIs are configured to talk to local Floci emulators in the exercises; they do not require real cloud accounts. Never use production credentials with these examples.

### AWS CLI v2

Needed for AWS/Floci commands such as `aws ec2 run-instances`, as well as AWS CLI waiters and binary-payload options.

```bash
case "$(uname -m)" in
  x86_64) AWS_CLI_ARCH=x86_64 ;;
  aarch64|arm64) AWS_CLI_ARCH=aarch64 ;;
  *) echo "Unsupported AWS CLI architecture: $(uname -m)" >&2; exit 1 ;;
esac
curl -fsSLo /tmp/awscliv2.zip \
  "https://awscli.amazonaws.com/awscli-exe-linux-${AWS_CLI_ARCH}.zip"
unzip -q /tmp/awscliv2.zip -d /tmp
sudo /tmp/aws/install --update
rm -rf /tmp/aws /tmp/awscliv2.zip
```

Verify:

```bash
aws --version
aws ec2 run-instances --generate-cli-skeleton input >/dev/null
```

### Azure CLI

Needed for the Floci AZ data-plane exercises. Do not run `az login` for the local emulator labs.

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://packages.microsoft.com/keys/microsoft.asc \
  | sudo gpg --dearmor -o /etc/apt/keyrings/microsoft.gpg
AZ_REPO=$(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
echo "Types: deb
URIs: https://packages.microsoft.com/repos/azure-cli/
Suites: ${AZ_REPO}
Components: main
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/microsoft.gpg" \
  | sudo tee /etc/apt/sources.list.d/azure-cli.sources
sudo apt-get update
sudo apt-get install -y azure-cli
```

Verify:

```bash
az --version
```

### Google Cloud CLI (`gcloud`)

Needed for Floci GCP CLI exercises. Do not run `gcloud auth login` for the local emulator labs.

```bash
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg \
  | sudo gpg --dearmor -o /usr/share/keyrings/cloud.google.gpg
echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" \
  | sudo tee /etc/apt/sources.list.d/google-cloud-sdk.list
sudo apt-get update
sudo apt-get install -y google-cloud-cli
```

Verify:

```bash
gcloud --version
```

### kubectl

Needed for Kubernetes and Minikube labs.

```bash
curl -fsSLo /tmp/kubectl https://dl.k8s.io/release/$(curl -fsSL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl
sudo install -o root -g root -m 0755 /tmp/kubectl /usr/local/bin/kubectl
rm /tmp/kubectl
```

Verify:

```bash
kubectl version --client
```

### Minikube

Needed to run a local Kubernetes cluster.

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64
```

Verify:

```bash
minikube version
```

### Terraform

Needed for the AWS, Azure, and GCP IaC exercises.

```bash
wget -O- https://apt.releases.hashicorp.com/gpg \
  | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(. /etc/os-release && echo "$VERSION_CODENAME") main" \
  | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt-get update
sudo apt-get install -y terraform
```

Verify:

```bash
terraform version
```

### Ansible

Needed for the host-configuration exercise.

```bash
sudo apt-get update
sudo apt-get install -y ansible
```

Verify:

```bash
ansible --version
```

### Docker and Git

Docker and Git are shared prerequisites for the container, simulator, and source-control labs. Docker Engine installation is described above.

```bash
git --version
docker version
docker compose version
```

These installation commands download software from upstream repositories. For a managed or production host, follow your organization's package pinning, verification, and change-management process.

**Debug:** The last printed `Step` marker identifies the setup phase to retry. Check network and package-manager health with `sudo apt-get update`, `df -h`, and `systemctl --no-pager status docker`; after adding your user to the Docker group, start a new login session before testing Docker access.

---
