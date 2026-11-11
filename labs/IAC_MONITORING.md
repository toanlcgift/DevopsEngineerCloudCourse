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

## Terraform: Provision S3 in LocalStack and Floci

Use the same AWS-provider configuration against either local AWS simulator. Start only one simulator at a time: both use port 4566. Follow the startup and authentication instructions in the [AWS simulator lab](CLOUD_DEVSECOPS.md).

```bash
echo "Step 1/3: Create a Terraform project for the S3 bucket"
mkdir -p ~/devops-labs/terraform-s3-simulator
cd ~/devops-labs/terraform-s3-simulator
cat > main.tf <<'EOF'
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

variable "simulator_endpoint" {
  type        = string
  description = "Local AWS-compatible endpoint"
}

provider "aws" {
  region                      = "us-east-1"
  access_key                  = "test"
  secret_key                  = "test"
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true
  s3_use_path_style           = true

  endpoints {
    s3 = var.simulator_endpoint
  }
}

resource "aws_s3_bucket" "artifacts" {
  bucket = "iac-lab-artifacts"

  tags = {
    Environment = "lab"
    ManagedBy   = "terraform"
  }
}

resource "aws_s3_bucket_public_access_block" "artifacts" {
  bucket                  = aws_s3_bucket.artifacts.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_server_side_encryption_configuration" "artifacts" {
  bucket = aws_s3_bucket.artifacts.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
EOF

echo "Step 2/3: Initialize, review the plan, and apply to the running simulator"
terraform init
terraform fmt -check
terraform validate
terraform plan -var='simulator_endpoint=http://127.0.0.1:4566'
terraform apply -var='simulator_endpoint=http://127.0.0.1:4566'

echo "Step 3/3: Verify the bucket, then remove the simulator resources"
aws --endpoint-url http://127.0.0.1:4566 s3 ls
terraform destroy -var='simulator_endpoint=http://127.0.0.1:4566'
```

Run the apply, verification, and destroy sequence once with LocalStack and once with Floci, stopping one simulator before starting the other. Compare the Terraform plan and AWS CLI output. The credentials in this provider are fake and must only be used with the simulator endpoint; Floci and LocalStack may differ from AWS in service behavior. Keep the Terraform state and `.terraform` directory out of source control, and review plans before applying.

**Debug:** Check the selected simulator's health endpoint and container logs in the cloud simulator lab. Confirm the provider endpoint is `http://127.0.0.1:4566`, that only one simulator owns port 4566, and that Terraform has not been pointed at a real AWS endpoint.

## Terraform: Provision a Durable SQS Work Queue

This lab provisions a queue and dead-letter queue against either LocalStack or Floci. Start one simulator and use the same local endpoint configuration shown in the [AWS simulator labs](CLOUD_DEVSECOPS.md).

```bash
echo "Step 1/3: Create the Terraform SQS project"
mkdir -p ~/devops-labs/terraform-sqs-simulator
cd ~/devops-labs/terraform-sqs-simulator
cat > main.tf <<'EOF'
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

variable "simulator_endpoint" {
  type        = string
  description = "Local AWS-compatible endpoint"
}

provider "aws" {
  region                      = "us-east-1"
  access_key                  = "test"
  secret_key                  = "test"
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true

  endpoints {
    sqs = var.simulator_endpoint
  }
}

resource "aws_sqs_queue" "dead_letter" {
  name                      = "iac-lab-jobs-dead-letter"
  message_retention_seconds = 1209600
  sqs_managed_sse_enabled   = true
}

resource "aws_sqs_queue" "jobs" {
  name                       = "iac-lab-jobs"
  visibility_timeout_seconds = 30
  message_retention_seconds  = 86400
  sqs_managed_sse_enabled    = true
  redrive_policy             = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.dead_letter.arn
    maxReceiveCount     = 3
  })
}

output "jobs_queue_url" {
  value = aws_sqs_queue.jobs.url
}
EOF

echo "Step 2/3: Validate and apply to the local simulator"
terraform init
terraform fmt -check
terraform validate
terraform plan -var='simulator_endpoint=http://127.0.0.1:4566'
terraform apply -var='simulator_endpoint=http://127.0.0.1:4566'

echo "Step 3/3: Send a test message, inspect the queue, and clean up"
QUEUE_URL=$(terraform output -raw jobs_queue_url)
aws --endpoint-url http://127.0.0.1:4566 sqs send-message \
  --queue-url "$QUEUE_URL" --message-body '{"job":"test"}'
aws --endpoint-url http://127.0.0.1:4566 sqs receive-message \
  --queue-url "$QUEUE_URL" --wait-time-seconds 1
terraform destroy -var='simulator_endpoint=http://127.0.0.1:4566'
```

Run this lab against LocalStack and Floci separately. The dead-letter queue receives messages after repeated unsuccessful processing; review the planned redrive policy before applying. These fake credentials must only be used with the loopback simulator endpoint. Simulator behavior is not a substitute for testing production delivery semantics or IAM policies.

**Debug:** Confirm the selected simulator is running on port 4566, inspect its logs, and check `terraform output -raw jobs_queue_url`. If a service endpoint error appears, verify the AWS provider's `sqs` endpoint and do not point this fake-credential configuration at AWS.

## Floci AWS Labs for the Cloud-Service Capability Matrix

The `floci` image emulates AWS APIs. Start it before running any AWS/Floci exercise below. This publishes its gateway only on localhost port 4566:

```bash
docker run -d --name floci \
  -p 127.0.0.1:4566:4566 \
  floci/floci:latest
docker ps --filter name=floci
docker logs --tail 50 floci
```

In the same terminal, configure the AWS CLI to use Floci with fake credentials:

```bash
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1
export AWS_ENDPOINT_URL=http://127.0.0.1:4566
aws --endpoint-url "$AWS_ENDPOINT_URL" sts get-caller-identity
```

Stop the simulator when finished:

```bash
docker rm -f floci
```

The Azure and GCP labs use **separate Docker containers and images**; starting the AWS container above does not start them. Start each emulator with its own `docker compose up -d` command in that lab's directory. They can run alongside each other because their host ports differ:

| Emulator | Docker image | Local endpoint |
|---|---|---|
| AWS | `floci/floci` | `http://127.0.0.1:4566` |
| Azure | `floci/floci-az` | `http://127.0.0.1:4577` (HTTPS is also enabled for Terraform) |
| GCP | `floci/floci-gcp` | `http://127.0.0.1:4588` |

Each lab has its own `compose.yaml`, startup, and cleanup commands. The Azure Terraform provider uses the Azure emulator's HTTPS metadata endpoint; the GCP provider uses the GCP emulator's custom API endpoints. Do not send one cloud provider's CLI or Terraform provider to another emulator's port.

| AWS capability | Floci AWS equivalent | Lab |
|---|---|---|
| Object storage | S3 | [Terraform S3 lab](#terraform-provision-s3-in-localstack-and-floci) |
| Virtual machines | EC2 instances (Docker-backed) | EC2 exercise below |
| Serverless | Lambda | Lambda exercise below |
| Containers | ECS tasks (Docker-backed) | ECS exercise below |
| Relational database | RDS | RDS exercise below |
| NoSQL | DynamoDB | DynamoDB exercise below |
| Messaging | SQS and SNS | [Terraform SQS lab](#terraform-provision-a-durable-sqs-work-queue) and SNS exercise below |
| Secrets | Secrets Manager | Secrets exercise below |
| Encryption | KMS | KMS exercise below |
| Infrastructure as Code | Terraform and CloudFormation | Terraform labs above and CloudFormation exercise below |
| Monitoring | CloudWatch Logs | CloudWatch exercise below |
| Identity | IAM | IAM exercise below |

The following two labs use the dedicated Floci Azure and GCP emulators and native Terraform providers rather than AWS resource equivalents.

### Azure IaC: Azure Storage with Floci AZ

Start Floci AZ with TLS enabled. TLS is required by Terraform's `azurerm` provider. The Docker socket allows Floci AZ to run container-backed services; this storage exercise itself does not launch a guest container.

```bash
mkdir -p ~/devops-labs/floci-az
cd ~/devops-labs/floci-az
cat > compose.yaml <<'EOF'
services:
  floci-az:
    image: floci/floci-az:latest
    ports:
      - "127.0.0.1:4577:4577"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data:/app/data
    environment:
      FLOCI_AZ_TLS_ENABLED: "true"
EOF
docker compose up -d
docker compose ps
docker compose logs --tail 50 floci-az
curl -fsS http://localhost:4577/_floci/tls-cert -o floci-az.crt
sudo install -m 0644 floci-az.crt /usr/local/share/ca-certificates/floci-az.crt
sudo update-ca-certificates
```

In the same directory, configure Terraform to use the local Azure cloud and create a resource group, storage account, and blob container:

```bash
cat > main.tf <<'EOF'
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
  use_cli = false

  environment   = "stack"
  metadata_host = "localhost:4577"

  subscription_id = "00000000-0000-0000-0000-000000000001"
  tenant_id       = "00000000-0000-0000-0000-000000000002"
  client_id       = "11111111-1111-1111-1111-111111111111"
  client_secret   = "floci-az-dev-secret"
}

resource "azurerm_resource_group" "lab" {
  name     = "floci-iac-lab"
  location = "eastus"
}

resource "azurerm_storage_account" "lab" {
  name                     = "flociiaclabstore"
  resource_group_name      = azurerm_resource_group.lab.name
  location                 = azurerm_resource_group.lab.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

resource "azurerm_storage_container" "artifacts" {
  name                  = "artifacts"
  storage_account_name  = azurerm_storage_account.lab.name
  container_access_type = "private"
}

output "storage_account_name" {
  value = azurerm_storage_account.lab.name
}
EOF

terraform init
terraform fmt -check
terraform validate
terraform plan
terraform apply
terraform output
terraform destroy
```

For a data-plane follow-up, install the Azure CLI from [SETUP.md](SETUP.md) and use only the local Floci AZ endpoint/account values documented in the [Floci AZ storage guide](https://floci.io/floci-az/). Do not run `az login` or use real Azure credentials for this emulator exercise. The provider uses test IDs and a local development secret; never reuse them outside Floci AZ. For Azure CLI command practice, use the separate [Azurite exercise](CLOUD_DEVSECOPS.md#azure-blob-storage-simulator-with-azurite); Azurite and Floci AZ are different emulators with different setup and endpoint values.

#### Azure CLI cheat-sheet commands

These commands follow Microsoft's [Azure CLI onboarding cheat sheet](https://learn.microsoft.com/en-us/cli/azure/cheat-sheet-onboarding?view=azure-cli-latest). Run them only in a separate, authenticated Azure CLI session; they inspect your real account context and do not target Floci AZ:

```bash
az account show --output table
az account list --output table
az account show --query "{tenantId:tenantId,subscriptionid:id}" --output json
az group list --query "[].{Name:name, Location:location}" --output table
az vm list --show-details \
  --query "[].{Name:name, ResourceGroup:resourceGroup, PowerState:powerState}" \
  --output table
az storage account list \
  --query "[].{Name:name, ResourceGroup:resourceGroup, Location:location}" \
  --output table
az find "storage blob list"
```

These commands list subscription context, resource groups, VMs, and storage accounts. The `--query` expressions select fields from JSON before `--output table` renders them; try `--output json` to inspect the full response. `az find` searches for command examples. All commands are read-only but query your real subscription; run them only in a separate, authenticated Azure CLI session. Do not run `az login` as part of the local emulator exercise.

**Cleanup:** After Terraform destroy, stop the emulator with `docker compose down`. Remove the trusted lab certificate with `sudo rm /usr/local/share/ca-certificates/floci-az.crt && sudo update-ca-certificates`.

### GCP IaC: Cloud Storage and Pub/Sub with Floci GCP

Start the GCP emulator. It uses port 4588 and mounts the Docker socket for services with container-backed data planes:

```bash
mkdir -p ~/devops-labs/floci-gcp
cd ~/devops-labs/floci-gcp
cat > compose.yaml <<'EOF'
services:
  floci-gcp:
    image: floci/floci-gcp:latest
    ports:
      - "127.0.0.1:4588:4588"
    volumes:
      - ./data:/app/data
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      FLOCI_GCP_HOSTNAME: floci-gcp
      FLOCI_GCP_BASE_URL: http://floci-gcp:4588
      FLOCI_GCP_STORAGE_MODE: hybrid
EOF
docker compose up -d
docker compose ps
docker compose logs --tail 50 floci-gcp
export GOOGLE_CLOUD_PROJECT=floci-local
export STORAGE_EMULATOR_HOST=http://127.0.0.1:4588
export PUBSUB_EMULATOR_HOST=127.0.0.1:4588
export GOOGLE_OAUTH_ACCESS_TOKEN=test
```

Create a GCS bucket/object and Pub/Sub topic/subscription using Terraform's Google provider endpoints. The custom endpoints keep provider API calls on the local emulator:

```bash
cat > main.tf <<'EOF'
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 7.36"
    }
  }
}

variable "endpoint" {
  type    = string
  default = "http://127.0.0.1:4588"
}

provider "google" {
  project               = "floci-local"
  region                = "us-central1"
  user_project_override = false

  storage_custom_endpoint          = "${var.endpoint}/storage/v1/"
  pubsub_custom_endpoint           = "${var.endpoint}/v1/"
  resource_manager_custom_endpoint = "${var.endpoint}/v1/"
  service_usage_custom_endpoint    = "${var.endpoint}/v1/"
  cloud_billing_custom_endpoint    = "${var.endpoint}/v1/"
}

resource "google_storage_bucket" "artifacts" {
  name          = "floci-iac-lab-artifacts"
  location      = "US"
  force_destroy = true
}

resource "google_storage_bucket_object" "readme" {
  bucket  = google_storage_bucket.artifacts.name
  name    = "README.txt"
  content = "Created with Terraform against Floci GCP."
}

resource "google_pubsub_topic" "events" {
  name = "floci-iac-lab-events"
}

resource "google_pubsub_subscription" "worker" {
  name  = "floci-iac-lab-worker"
  topic = google_pubsub_topic.events.id
}
EOF

terraform init
terraform fmt -check
terraform validate
terraform plan
terraform apply
terraform output
```

For CLI practice, install `gcloud` from [SETUP.md](SETUP.md) and use the same terminal where the emulator variables were exported above. After `terraform apply` has created the bucket and `README.txt`, inspect the bucket, upload and list a sample object, then remove that object before `terraform destroy`:

```bash
gcloud storage --help
gcloud storage ls --help
gcloud storage buckets list --format='table(name)'
gcloud storage ls --long gs://floci-iac-lab-artifacts

printf 'gcloud CLI practice\n' > cli-note.txt
gcloud storage cp cli-note.txt gs://floci-iac-lab-artifacts/cli-note.txt
gcloud storage ls --format=json gs://floci-iac-lab-artifacts
gcloud storage rm gs://floci-iac-lab-artifacts/cli-note.txt
```

Compare the default output with `--format=json`, and use `gcloud storage ls --help` to explore options. For more command patterns and examples, see Google's [gcloud CLI cheat sheet](https://docs.cloud.google.com/sdk/docs/cheatsheet). It describes the command tree and global flags such as `--format`; try `gcloud help storage` to search command reference material and `gcloud topic formats` to learn about output formats. These examples use Floci GCP's Cloud Storage emulator; they do not demonstrate Cloud Monitoring metrics or alerting. Do not run `gcloud auth login` or use real GCP credentials. Validate emulator behavior against real GCP separately before production use.

#### Google Cloud CLI cheat-sheet commands

The commands below are examples from Google's [gcloud CLI cheat sheet](https://docs.cloud.google.com/sdk/docs/cheatsheet). Unlike the emulator commands above, they query real Google Cloud and require an authenticated account with permission to view the selected project. Substitute a project you are authorized to inspect; these commands are read-only:

```bash
gcloud version
gcloud info
gcloud config list

PROJECT_ID="your-project-id"
gcloud projects list --format="table(projectId,name)"
gcloud projects describe "$PROJECT_ID"
gcloud logging logs list --project="$PROJECT_ID"
gcloud compute zones list --project="$PROJECT_ID" --limit=10 \
  --format="table(name,region.basename())"
gcloud compute instances list --project="$PROJECT_ID" \
  --format="table(name,zone,status)"
gcloud compute instances list --project="$PROJECT_ID" \
  --filter="status=RUNNING" --limit=10 \
  --format="table(name,zone,status)"
gcloud container clusters list --project="$PROJECT_ID" \
  --format="table(name,location,status)"
```

The first commands inspect your local CLI setup; the others list accessible projects, project metadata, logs, zones, VMs, and GKE clusters. Compare the unfiltered and filtered VM lists to see how `--filter`, `--limit`, and `--format` shape results. These commands do not use Floci GCP and do not create resources; they require an authenticated account and appropriate read permissions. They are inventory/log commands, not Cloud Monitoring metric or alert queries.

**Cleanup:** Run `terraform destroy`, then stop Floci GCP with `docker compose down`.

Floci runs EC2 instances, ECS tasks, Lambda functions, and RDS databases in local Docker containers. Docker must be running; these exercises can pull images, use disk, and consume CPU and memory. EC2 containers can run privileged when Floci's EC2 security-group enforcement is disabled; use a trusted, disposable lab machine and image, never an untrusted guest or shared production Docker host.

### Virtual machines: EC2

Floci's EC2 instances run as Docker containers, not virtual machines with an independent guest kernel. Launch a small catalog image, inspect it, and terminate it:

```bash
INSTANCE_ID=$(aws --endpoint-url "$AWS_ENDPOINT_URL" ec2 run-instances \
  --image-id ami-alpine --instance-type t3.micro \
  --min-count 1 --max-count 1 \
  --query 'Instances[0].InstanceId' --output text)
aws --endpoint-url "$AWS_ENDPOINT_URL" ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].[InstanceId,ImageId,InstanceType,State.Name]' \
  --output table
aws --endpoint-url "$AWS_ENDPOINT_URL" ec2 terminate-instances \
  --instance-ids "$INSTANCE_ID"
aws --endpoint-url "$AWS_ENDPOINT_URL" ec2 wait instance-terminated \
  --instance-ids "$INSTANCE_ID"
```

**Try it:** Compare the instance's reported state before and after termination. Explain which VM properties are only approximated by a container-backed emulator.

### Serverless: Lambda

Create a small Python handler, package it, create a local execution role, invoke the function, and remove both resources:

```bash
mkdir -p ~/devops-labs/floci-lambda
cd ~/devops-labs/floci-lambda
cat > function.py <<'EOF'
def handler(event, context):
    return {"message": "hello from Floci", "input": event}
EOF
python3 -c 'import zipfile; z=zipfile.ZipFile("function.zip","w"); z.write("function.py"); z.close()'
cat > trust-policy.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF
ROLE_ARN=$(aws --endpoint-url "$AWS_ENDPOINT_URL" iam create-role \
  --role-name floci-lambda-lab \
  --assume-role-policy-document file://trust-policy.json \
  --query 'Role.Arn' --output text)
aws --endpoint-url "$AWS_ENDPOINT_URL" lambda create-function \
  --function-name floci-hello \
  --runtime python3.12 --handler function.handler \
  --role "$ROLE_ARN" --zip-file fileb://function.zip
aws --endpoint-url "$AWS_ENDPOINT_URL" lambda invoke \
  --function-name floci-hello \
  --payload '{"course":"devops"}' \
  --cli-binary-format raw-in-base64-out response.json
cat response.json
aws --endpoint-url "$AWS_ENDPOINT_URL" lambda delete-function \
  --function-name floci-hello
aws --endpoint-url "$AWS_ENDPOINT_URL" iam delete-role \
  --role-name floci-lambda-lab
```

**Try it:** Change the input JSON and return one additional field. Lambda execution is container-based in Floci and is not the AWS Lambda Firecracker isolation model.

### Containers: ECS task

This Fargate-compatible task runs a short-lived BusyBox container locally. The exercise creates a private VPC and subnet for the task's `awsvpc` networking; it does not open inbound ports:

```bash
VPC_ID=$(aws --endpoint-url "$AWS_ENDPOINT_URL" ec2 create-vpc \
  --cidr-block 10.88.0.0/16 --query 'Vpc.VpcId' --output text)
SUBNET_ID=$(aws --endpoint-url "$AWS_ENDPOINT_URL" ec2 create-subnet \
  --vpc-id "$VPC_ID" --cidr-block 10.88.1.0/24 \
  --query 'Subnet.SubnetId' --output text)
SG_ID=$(aws --endpoint-url "$AWS_ENDPOINT_URL" ec2 create-security-group \
  --group-name floci-ecs-lab --description 'Floci ECS lab' \
  --vpc-id "$VPC_ID" --query GroupId --output text)
aws --endpoint-url "$AWS_ENDPOINT_URL" ecs create-cluster \
  --cluster-name floci-lab
cat > task-definition.json <<'EOF'
{
  "family": "floci-hello",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [{
    "name": "hello",
    "image": "public.ecr.aws/docker/library/busybox:1.36",
    "essential": true,
    "command": ["sh", "-c", "echo hello-from-ecs; sleep 2"]
  }]
}
EOF
aws --endpoint-url "$AWS_ENDPOINT_URL" ecs register-task-definition \
  --cli-input-json file://task-definition.json
TASK_ARN=$(aws --endpoint-url "$AWS_ENDPOINT_URL" ecs run-task \
  --cluster floci-lab --launch-type FARGATE \
  --task-definition floci-hello \
  --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_ID],securityGroups=[$SG_ID],assignPublicIp=DISABLED}" \
  --query 'tasks[0].taskArn' --output text)
aws --endpoint-url "$AWS_ENDPOINT_URL" ecs wait tasks-stopped \
  --cluster floci-lab --tasks "$TASK_ARN"
aws --endpoint-url "$AWS_ENDPOINT_URL" ecs describe-tasks \
  --cluster floci-lab --tasks "$TASK_ARN" \
  --query 'tasks[0].[lastStatus,stoppedReason]' --output table
aws --endpoint-url "$AWS_ENDPOINT_URL" ecs deregister-task-definition \
  --task-definition floci-hello
aws --endpoint-url "$AWS_ENDPOINT_URL" ecs delete-cluster \
  --cluster floci-lab
aws --endpoint-url "$AWS_ENDPOINT_URL" ec2 delete-security-group \
  --group-id "$SG_ID"
aws --endpoint-url "$AWS_ENDPOINT_URL" ec2 delete-subnet \
  --subnet-id "$SUBNET_ID"
aws --endpoint-url "$AWS_ENDPOINT_URL" ec2 delete-vpc --vpc-id "$VPC_ID"
```

**Try it:** Add a second container that prints a different message, then inspect both container exit statuses. ECS tasks in Floci use Docker; this does not create an AWS-managed ECS cluster or validate EKS/Kubernetes behavior.

### Relational database: RDS

RDS starts a local database container. Use this fake password only in the isolated lab:

```bash
aws --endpoint-url "$AWS_ENDPOINT_URL" rds create-db-instance \
  --db-instance-identifier floci-lab-db \
  --db-instance-class db.t3.micro \
  --engine postgres --master-username labuser \
  --master-user-password 'LabOnly-Replace-123!' \
  --allocated-storage 20 --backup-retention-period 0
aws --endpoint-url "$AWS_ENDPOINT_URL" rds wait db-instance-available \
  --db-instance-identifier floci-lab-db
aws --endpoint-url "$AWS_ENDPOINT_URL" rds describe-db-instances \
  --db-instance-identifier floci-lab-db \
  --query 'DBInstances[0].[DBInstanceStatus,Engine,Endpoint.Address,Endpoint.Port]' \
  --output table
aws --endpoint-url "$AWS_ENDPOINT_URL" rds delete-db-instance \
  --db-instance-identifier floci-lab-db --skip-final-snapshot
aws --endpoint-url "$AWS_ENDPOINT_URL" rds wait db-instance-deleted \
  --db-instance-identifier floci-lab-db
```

**Try it:** Check the returned engine and endpoint metadata. If you have a PostgreSQL client installed, connect only to the local emulator endpoint and create a disposable table. RDS does not emulate Azure SQL or Cloud SQL-specific features.

### NoSQL: DynamoDB

Create a table, write and read a JSON item, then delete the table:

```bash
aws --endpoint-url "$AWS_ENDPOINT_URL" dynamodb create-table \
  --table-name floci-lab-items \
  --attribute-definitions AttributeName=pk,AttributeType=S \
  --key-schema AttributeName=pk,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
aws --endpoint-url "$AWS_ENDPOINT_URL" dynamodb wait table-exists \
  --table-name floci-lab-items
aws --endpoint-url "$AWS_ENDPOINT_URL" dynamodb put-item \
  --table-name floci-lab-items \
  --item '{"pk":{"S":"lab-1"},"status":{"S":"ready"},"attempts":{"N":"1"}}'
aws --endpoint-url "$AWS_ENDPOINT_URL" dynamodb get-item \
  --table-name floci-lab-items --key '{"pk":{"S":"lab-1"}}'
aws --endpoint-url "$AWS_ENDPOINT_URL" dynamodb delete-table \
  --table-name floci-lab-items
```

**Try it:** Update the item's status with `update-item` and explain how a DynamoDB partition key differs from a relational primary key.

### Messaging: SNS publish/subscribe

Create a topic and publish a harmless event:

```bash
TOPIC_ARN=$(aws --endpoint-url "$AWS_ENDPOINT_URL" sns create-topic \
  --name floci-lab-events --query TopicArn --output text)
aws --endpoint-url "$AWS_ENDPOINT_URL" sns list-topics
aws --endpoint-url "$AWS_ENDPOINT_URL" sns publish \
  --topic-arn "$TOPIC_ARN" \
  --message '{"event":"build-finished","status":"success"}'
```

**Try it:** Add an SQS subscription to the topic and receive the published notification:

```bash
QUEUE_URL=$(aws --endpoint-url "$AWS_ENDPOINT_URL" sqs create-queue \
  --queue-name floci-lab-events --query QueueUrl --output text)
QUEUE_ARN=$(aws --endpoint-url "$AWS_ENDPOINT_URL" sqs get-queue-attributes \
  --queue-url "$QUEUE_URL" --attribute-names QueueArn \
  --query Attributes.QueueArn --output text)
cat > sns-queue-policy.json <<EOF
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"sns.amazonaws.com"},"Action":"sqs:SendMessage","Resource":"$QUEUE_ARN","Condition":{"ArnEquals":{"aws:SourceArn":"$TOPIC_ARN"}}}]}
EOF
python3 -c 'import json; p=json.load(open("sns-queue-policy.json")); json.dump({"Policy":json.dumps(p)},open("sns-queue-attributes.json","w"))'
aws --endpoint-url "$AWS_ENDPOINT_URL" sqs set-queue-attributes \
  --queue-url "$QUEUE_URL" --attributes file://sns-queue-attributes.json
SUBSCRIPTION_ARN=$(aws --endpoint-url "$AWS_ENDPOINT_URL" sns subscribe \
  --topic-arn "$TOPIC_ARN" --protocol sqs --notification-endpoint "$QUEUE_ARN" \
  --query SubscriptionArn --output text)
aws --endpoint-url "$AWS_ENDPOINT_URL" sns publish \
  --topic-arn "$TOPIC_ARN" --message '{"event":"queue-check"}'
aws --endpoint-url "$AWS_ENDPOINT_URL" sqs receive-message \
  --queue-url "$QUEUE_URL" --wait-time-seconds 1
aws --endpoint-url "$AWS_ENDPOINT_URL" sns unsubscribe \
  --subscription-arn "$SUBSCRIPTION_ARN"
aws --endpoint-url "$AWS_ENDPOINT_URL" sqs delete-queue \
  --queue-url "$QUEUE_URL"
aws --endpoint-url "$AWS_ENDPOINT_URL" sns delete-topic \
  --topic-arn "$TOPIC_ARN"
```

Use the existing [SQS Terraform lab](#terraform-provision-a-durable-sqs-work-queue) to explore queue provisioning. SNS/SQS emulator success does not prove real AWS delivery policy or permissions.

### Secrets: Secrets Manager

Store and read an explicitly fake value. Do not substitute any real credential:

```bash
SECRET_ARN=$(aws --endpoint-url "$AWS_ENDPOINT_URL" secretsmanager create-secret \
  --name floci-lab-demo \
  --secret-string '{"apiKey":"not-a-real-key"}' \
  --query ARN --output text)
aws --endpoint-url "$AWS_ENDPOINT_URL" secretsmanager describe-secret \
  --secret-id "$SECRET_ARN"
aws --endpoint-url "$AWS_ENDPOINT_URL" secretsmanager get-secret-value \
  --secret-id "$SECRET_ARN" --query SecretString --output text
aws --endpoint-url "$AWS_ENDPOINT_URL" secretsmanager delete-secret \
  --secret-id "$SECRET_ARN" --force-delete-without-recovery
```

**Try it:** Create a new secret version with `put-secret-value`, then inspect its version stages. The returned value is for demonstration only; do not print production secrets in a terminal or pipeline log.

### Encryption: KMS

Encrypt and decrypt a public sample sentence with a local symmetric key:

```bash
printf 'public lab sample, not a secret\n' > plaintext.txt
KEY_ID=$(aws --endpoint-url "$AWS_ENDPOINT_URL" kms create-key \
  --description 'Floci training key' \
  --key-usage ENCRYPT_DECRYPT --key-spec SYMMETRIC_DEFAULT \
  --query KeyMetadata.KeyId --output text)
aws --endpoint-url "$AWS_ENDPOINT_URL" kms encrypt \
  --key-id "$KEY_ID" --plaintext fileb://plaintext.txt \
  --query CiphertextBlob --output text > ciphertext.b64
base64 --decode ciphertext.b64 > ciphertext.bin
aws --endpoint-url "$AWS_ENDPOINT_URL" kms decrypt \
  --ciphertext-blob fileb://ciphertext.bin \
  --query Plaintext --output text | base64 --decode
aws --endpoint-url "$AWS_ENDPOINT_URL" kms schedule-key-deletion \
  --key-id "$KEY_ID" --pending-window-in-days 7
```

**Try it:** Compare the ciphertext and recovered plaintext. A simulator does not provide production key custody, compliance guarantees, or a substitute for a real KMS key.

### Infrastructure as Code: CloudFormation

Terraform examples appear earlier in this file. This separate exercise deploys a queue from a CloudFormation template and removes the stack:

```bash
cat > queue-stack.json <<'EOF'
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Resources": {
    "Jobs": {
      "Type": "AWS::SQS::Queue",
      "Properties": {
        "QueueName": "floci-cfn-jobs",
        "VisibilityTimeout": 30
      }
    }
  },
  "Outputs": {
    "QueueUrl": {
      "Value": {"Ref": "Jobs"}
    }
  }
}
EOF
aws --endpoint-url "$AWS_ENDPOINT_URL" cloudformation create-stack \
  --stack-name floci-cfn-lab --template-body file://queue-stack.json
aws --endpoint-url "$AWS_ENDPOINT_URL" cloudformation wait stack-create-complete \
  --stack-name floci-cfn-lab
aws --endpoint-url "$AWS_ENDPOINT_URL" cloudformation describe-stacks \
  --stack-name floci-cfn-lab \
  --query 'Stacks[0].Outputs'
aws --endpoint-url "$AWS_ENDPOINT_URL" cloudformation delete-stack \
  --stack-name floci-cfn-lab
aws --endpoint-url "$AWS_ENDPOINT_URL" cloudformation wait stack-delete-complete \
  --stack-name floci-cfn-lab
```

**Try it:** Change the queue visibility timeout, create a change set, and inspect its diff before execution. Floci supports selected CloudFormation resources and operations; it is not a full replacement for AWS template validation.

### Monitoring: CloudWatch Logs

Create a log group and stream, write normal and error events, query the logs and metric, and clean up:

```bash
aws --endpoint-url "$AWS_ENDPOINT_URL" logs create-log-group \
  --log-group-name /devops-lab/app
aws --endpoint-url "$AWS_ENDPOINT_URL" logs create-log-stream \
  --log-group-name /devops-lab/app --log-stream-name local
TIMESTAMP=$(date +%s%3N)
aws --endpoint-url "$AWS_ENDPOINT_URL" logs put-log-events \
  --log-group-name /devops-lab/app --log-stream-name local \
  --log-events "[{\"timestamp\":$TIMESTAMP,\"message\":\"lab application ready\"}]"
ERROR_TIMESTAMP=$(date +%s%3N)
aws --endpoint-url "$AWS_ENDPOINT_URL" logs put-log-events \
  --log-group-name /devops-lab/app --log-stream-name local \
  --log-events "[{\"timestamp\":$ERROR_TIMESTAMP,\"message\":\"ERROR simulated failure for CLI practice\"}]"
aws --endpoint-url "$AWS_ENDPOINT_URL" logs get-log-events \
  --log-group-name /devops-lab/app --log-stream-name local
aws --endpoint-url "$AWS_ENDPOINT_URL" logs describe-log-groups \
  --log-group-name-prefix /devops-lab \
  --query 'logGroups[].logGroupName' --output table
aws --endpoint-url "$AWS_ENDPOINT_URL" logs filter-log-events \
  --log-group-name /devops-lab/app --filter-pattern '"ERROR"' \
  --query 'events[].message' --output text
aws --endpoint-url "$AWS_ENDPOINT_URL" cloudwatch put-metric-data \
  --namespace DevOpsLab --metric-name BuildSucceeded --value 1 --unit Count
aws --endpoint-url "$AWS_ENDPOINT_URL" cloudwatch list-metrics \
  --namespace DevOpsLab --metric-name BuildSucceeded \
  --query 'Metrics[].MetricName' --output table
aws --endpoint-url "$AWS_ENDPOINT_URL" cloudwatch get-metric-statistics \
  --namespace DevOpsLab --metric-name BuildSucceeded \
  --start-time "$(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%SZ)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --period 60 --statistics Sum
aws --endpoint-url "$AWS_ENDPOINT_URL" logs delete-log-group \
  --log-group-name /devops-lab/app
```

**Study the CLI:** Before running the lab, inspect a command's built-in help and its expected input shape:

```bash
AWS_PAGER="" aws logs help
aws logs put-log-events --generate-cli-skeleton input
```

In the monitoring commands, `--query` selects fields from the JSON response and `--output table` or `--output text` changes how the result is displayed. Try changing the filter pattern to `ready`, removing `--query`, or adding `--output json` to compare results. The `ERROR` message is synthetic test data; remove it with the log group during cleanup. These examples use AWS CLI against the local Floci emulator and demonstrate CloudWatch Logs/metrics commands, not Azure Monitor or Google Cloud Monitoring alerting behavior.

### Identity: IAM

Create a local role with a trust policy and an inline policy limited to one training queue; inspect and remove it:

```bash
cat > trust-policy.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF
ACCOUNT_ID=$(aws --endpoint-url "$AWS_ENDPOINT_URL" sts get-caller-identity \
  --query Account --output text)
cat > queue-policy.json <<EOF
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":["sqs:SendMessage"],"Resource":"arn:aws:sqs:us-east-1:$ACCOUNT_ID:floci-lab-jobs"}]}
EOF
aws --endpoint-url "$AWS_ENDPOINT_URL" iam create-role \
  --role-name floci-queue-producer \
  --assume-role-policy-document file://trust-policy.json
aws --endpoint-url "$AWS_ENDPOINT_URL" iam put-role-policy \
  --role-name floci-queue-producer \
  --policy-name send-to-training-queue \
  --policy-document file://queue-policy.json
aws --endpoint-url "$AWS_ENDPOINT_URL" iam get-role \
  --role-name floci-queue-producer
aws --endpoint-url "$AWS_ENDPOINT_URL" iam get-role-policy \
  --role-name floci-queue-producer \
  --policy-name send-to-training-queue
aws --endpoint-url "$AWS_ENDPOINT_URL" iam delete-role-policy \
  --role-name floci-queue-producer \
  --policy-name send-to-training-queue
aws --endpoint-url "$AWS_ENDPOINT_URL" iam delete-role \
  --role-name floci-queue-producer
```

**Try it:** Change the policy resource to a different queue ARN and explain why a wildcard resource is broader access. Floci can store and evaluate IAM resources when its IAM enforcement is configured, but this lab does not validate Azure Entra ID/RBAC or Google Cloud IAM behavior.

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
