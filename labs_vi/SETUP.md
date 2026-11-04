# Chuẩn bị Ubuntu Lab Host

Sử dụng Ubuntu Server 22.04 hoặc 24.04. Cấu hình VM khởi điểm hợp lý là 4 vCPU, 8 GB RAM và 30 GB dung lượng đĩa trống; Minikube và Jenkins chạy cùng lúc có thể cần nhiều hơn. Đăng nhập bằng user thông thường có quyền `sudo`.

Cập nhật host và cài đặt các command-line tools thông dụng:

```bash
echo "Step 1/2: Update apt and install common tools"
sudo apt-get update
sudo apt-get install -y ca-certificates curl git jq make python3 python3-pip \
  python3-venv unzip gnupg lsb-release openssl
echo "Step 2/2: Create the lab workspace"
mkdir -p ~/devops-labs
cd ~/devops-labs
```

Cài Docker Engine từ Ubuntu repository chính thức của Docker, sau đó cấp quyền chạy Docker cho user hiện tại. Thành viên nhóm Docker về thực tế có quyền tương đương root; chỉ thực hiện trên lab host đáng tin cậy. Sau lệnh `usermod`, hãy đăng xuất rồi đăng nhập lại, sau đó kiểm tra bằng `docker run --rm hello-world`.

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

Cài các Kubernetes và Cloud CLI sẽ sử dụng trong những lab tiếp theo:

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

Kiểm tra các phần mềm đã cài đặt:

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

Các lệnh cài đặt này tải phần mềm từ upstream repository. Với managed host hoặc production host, hãy tuân theo quy trình pin version package, xác minh và change management của tổ chức.

**Troubleshooting:** Dấu mốc `Step` cuối cùng được in ra cho biết giai đoạn setup cần thử lại. Kiểm tra mạng và package manager bằng `sudo apt-get update`, `df -h` và `systemctl --no-pager status docker`; sau khi thêm user vào nhóm Docker, hãy bắt đầu phiên đăng nhập mới trước khi kiểm tra quyền truy cập Docker.

---
