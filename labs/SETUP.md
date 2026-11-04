# Ubuntu Lab Host Setup

Use Ubuntu Server 22.04 or 24.04. A VM with 4 vCPUs, 8 GB RAM, and 30 GB free disk is a reasonable starting point; Minikube and Jenkins together may need more. Log in as a regular user with `sudo`.

Update the host and install common command-line tools:

```bash
echo "Step 1/2: Update apt and install common tools"
sudo apt-get update
sudo apt-get install -y ca-certificates curl git jq make python3 python3-pip \
  python3-venv unzip gnupg lsb-release
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

Install the Kubernetes and cloud CLIs used in later labs:

```bash
echo "Step 1/4: Install kubectl"
curl -fsSLo /tmp/kubectl https://dl.k8s.io/release/$(curl -fsSL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl
sudo install -o root -g root -m 0755 /tmp/kubectl /usr/local/bin/kubectl
rm /tmp/kubectl

echo "Step 2/4: Install Minikube"
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64

echo "Step 3/4: Add HashiCorp's repository and install Terraform"
wget -O- https://apt.releases.hashicorp.com/gpg \
  | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(. /etc/os-release && echo "$VERSION_CODENAME") main" \
  | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt-get update
sudo apt-get install -y terraform

echo "Step 4/4: Install Ansible"
sudo apt-get install -y ansible
```

Check the installation:

```bash
echo "Checking Docker and Compose"
docker version
docker compose version
echo "Checking Git and Kubernetes tools"
git --version
kubectl version --client
minikube version
echo "Checking Terraform and Ansible"
terraform version
ansible --version
```

These installation commands download software from upstream repositories. For a managed or production host, follow your organization's package pinning, verification, and change-management process.

**Debug:** The last printed `Step` marker identifies the setup phase to retry. Check network and package-manager health with `sudo apt-get update`, `df -h`, and `systemctl --no-pager status docker`; after adding your user to the Docker group, start a new login session before testing Docker access.

---
