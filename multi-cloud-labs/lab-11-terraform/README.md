# Lab 11 – Infrastructure as Code with Terraform

Use Terraform's local provider to manage a file, inspect state, change a variable, and destroy the resource. This is safe to run locally and creates no cloud infrastructure. Complete the [WSL setup](../README.md#start-here).

## 1. Install Terraform

```bash
echo "Step 1/2: Add HashiCorp's Ubuntu package repository"
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(. /etc/os-release && echo "$VERSION_CODENAME") main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
echo "Step 2/2: Install and verify Terraform"
sudo apt-get update
sudo apt-get install -y terraform
terraform version
```

## 2. Create a local Terraform configuration

```bash
mkdir -p ~/devops-labs/lab11
cd ~/devops-labs/lab11
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
```

## 3. Plan, apply, inspect, and destroy

```bash
terraform init
terraform fmt -check
terraform validate
terraform plan -var='environment=development' -out=tfplan
terraform show tfplan
terraform apply tfplan
cat generated.txt
terraform state list
terraform output artifact_path
terraform destroy -var='environment=development'
```

Terraform state and plan files can contain sensitive data. Keep `tfplan`, `.terraform/`, and `terraform.tfstate*` out of source control. For cloud providers, review every plan and configure remote state locking before collaborating; a local provider exercise does not validate provider IAM or cloud billing.

## 4. Clean up

```bash
cd ~/devops-labs/lab11
rm -rf .terraform .terraform.lock.hcl terraform.tfstate terraform.tfstate.backup tfplan generated.txt
```
