# Infrastructure as Code và giám sát

## Terraform: Quản lý artifact cục bộ

Bài thực hành Terraform đầu tiên này sử dụng provider `local` tích hợp sẵn nên không tạo tài nguyên đám mây.

```bash
echo "Step 1/3: Create the Terraform configuration"
mkdir -p ~/devops-labs/terraform-local
cd ~/devops-labs/terraform-local
cat > main.tf <<'EOF'
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.5"
    }
  }
}

variable "environment" {
  type    = string
  default = "lab"
}

resource "local_file" "inventory" {
  filename = "${path.module}/generated.txt"
  content  = "environment=${var.environment}\nmanaged_by=terraform\n"
}

output "artifact_path" {
  value = local_file.inventory.filename
}
EOF

echo "Step 2/3: Initialize and validate the configuration"
terraform init
terraform fmt -check
terraform validate
echo "Step 3/3: Plan, apply, inspect, and destroy the local artifact"
terraform plan -out=tfplan
terraform apply tfplan
cat generated.txt
terraform destroy
```

**Thử thực hành:** Truyền `-var='environment=test'` cho `terraform plan`; kiểm tra thay đổi được đề xuất trước khi áp dụng. Không bao giờ commit tệp state hoặc plan nếu chúng có thể chứa giá trị nhạy cảm.

**Gỡ lỗi:** Xem chính xác lỗi được in ra bởi `terraform init`, `validate` hoặc `apply`; sau đó chạy `terraform validate` và `terraform show tfplan` để kiểm tra cấu hình cùng các thay đổi đã lập kế hoạch. Để chẩn đoán ở cấp provider, chạy `TF_LOG=DEBUG terraform plan 2>terraform-debug.log`, rồi chạy `chmod 600 terraform-debug.log`; log gỡ lỗi có thể chứa dữ liệu nhạy cảm, vì vậy đừng chia sẻ nếu chưa xem xét.

## Ansible: Cấu hình máy chủ Ubuntu

```bash
echo "Step 1/3: Create the localhost inventory and playbook"
cd ~/devops-labs
mkdir -p ansible
cd ansible
cat > inventory.ini <<'EOF'
[local]
localhost ansible_connection=local
EOF
cat > site.yml <<'EOF'
---
- name: Configure the DevOps lab
  hosts: local
  become: true
  tasks:
    - name: Ensure the lab marker directory exists
      ansible.builtin.file:
        path: /opt/devops-lab
        state: directory
        mode: "0755"
EOF

echo "Step 2/3: Inspect inventory and run a check-mode preview"
ansible-inventory -i inventory.ini --list
ansible-playbook -i inventory.ini site.yml --check --diff
echo "Step 3/3: Apply the playbook and verify the directory"
ansible-playbook -i inventory.ini site.yml --diff --ask-become-pass
ansible localhost -i inventory.ini -m ansible.builtin.stat \
  -a 'path=/opt/devops-lab' -b --ask-become-pass
```

**Xác minh:** Play báo `changed` ở lần áp dụng đầu tiên và `ok` ở lần tiếp theo. Dùng `--check` trước khi thay đổi; khi thực hành trên nhiều máy, hãy giới hạn inventory ở các máy chủ có thể loại bỏ.

**Gỡ lỗi:** Thêm `-vvv` vào `ansible-playbook` để xem chi tiết kết nối và task. Khi xác minh inventory hoặc quyền hệ thống tệp, hãy kiểm tra `ansible-inventory -i inventory.ini --graph` và `sudo stat /opt/devops-lab`.

## Prometheus và Grafana: Metric và dashboard

```bash
echo "Step 1/3: Create Prometheus scrape configuration"
mkdir -p ~/devops-labs/monitoring
cd ~/devops-labs/monitoring
cat > prometheus.yml <<'EOF'
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["prometheus:9090"]
EOF
echo "Step 2/3: Define Prometheus and Grafana services"
cat > compose.yaml <<'EOF'
services:
  prometheus:
    image: prom/prometheus:v2.55.1
    command:
      - --config.file=/etc/prometheus/prometheus.yml
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports:
      - "127.0.0.1:9090:9090"
  grafana:
    image: grafana/grafana:11.3.0
    ports:
      - "127.0.0.1:3000:3000"
EOF

echo "Step 3/3: Start services and verify Prometheus health"
docker compose up -d
docker compose ps
curl --fail http://127.0.0.1:9090/-/healthy
```

Mở Grafana tại `http://127.0.0.1:3000` (thông tin đăng nhập mặc định của bài thực hành: `admin` / `admin`; hãy đổi khi được nhắc). Thêm data source Prometheus với URL `http://prometheus:9090`, sau đó truy vấn `up` trong Explore. Các service được bind vào loopback; hãy dùng SSH tunnel để truy cập từ xa.

**Thử thực hành:** Xem target của Prometheus tại `http://127.0.0.1:9090/targets`. Giải thích metric có thể cho biết điều gì mà một dòng log không thể hiện được, đồng thời cấu hình một panel Grafana đơn giản cho `up`.

**Gỡ lỗi:** Kiểm tra quá trình khởi động service bằng `docker compose ps` và `docker compose logs --tail 100 prometheus grafana`. Trong Prometheus, kiểm tra `/targets` để tìm lỗi scrape; từ máy chủ, chạy `curl -v http://127.0.0.1:9090/-/healthy`. Trang **Connections / Data sources** trong giao diện Grafana hiển thị lỗi kiểm tra datasource. Dấu mốc bước được in cuối cùng cho biết giai đoạn bị lỗi.

---
