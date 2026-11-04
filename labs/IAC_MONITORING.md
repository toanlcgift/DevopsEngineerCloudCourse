# Infrastructure as Code and Monitoring

## Terraform: Manage a Local Artifact

This first Terraform exercise uses the built-in `local` provider so no cloud resources are created.

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

**Try it:** Pass `-var='environment=test'` to `terraform plan`; inspect the proposed change before applying. Never commit state files or plans if they may contain sensitive values.

**Debug:** Review the exact failure printed by `terraform init`, `validate`, or `apply`; then run `terraform validate` and `terraform show tfplan` to inspect configuration and planned changes. For provider-level diagnostics, run `TF_LOG=DEBUG terraform plan 2>terraform-debug.log` and then `chmod 600 terraform-debug.log`; debug logs can contain sensitive data, so do not share them without review.

## Ansible: Configure the Ubuntu Host

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

**Verify:** The play reports `changed` on its first apply and `ok` on the next. Use `--check` before changes, and use an inventory limited to disposable hosts for fleet exercises.

**Debug:** Add `-vvv` to `ansible-playbook` to see connection and task details. Check `ansible-inventory -i inventory.ini --graph` and `sudo stat /opt/devops-lab` when verifying inventory or filesystem permissions.

## Prometheus and Grafana: Metrics and Dashboards

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

Open Grafana at `http://127.0.0.1:3000` (default lab login: `admin` / `admin`; change it when prompted). Add a Prometheus data source with URL `http://prometheus:9090`, then query `up` in Explore. The services are bound to loopback; use an SSH tunnel to access them remotely.

**Try it:** View Prometheus targets at `http://127.0.0.1:9090/targets`. Explain what a metric can show that a log line cannot, and configure a simple Grafana panel for `up`.

**Debug:** Check service startup with `docker compose ps` and `docker compose logs --tail 100 prometheus grafana`. In Prometheus, inspect `/targets` for scrape errors; from the host, check `curl -v http://127.0.0.1:9090/-/healthy`. Grafana's UI **Connections / Data sources** page reports datasource test failures. The last printed step marker points to the phase that failed.

---
