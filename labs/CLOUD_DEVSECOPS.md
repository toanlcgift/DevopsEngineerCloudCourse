# Cloud Simulators, DevSecOps, and Capstone

## AWS S3 Simulators: LocalStack and Floci

Run either AWS simulator locally. Both use port 4566, so stop one before starting the other. LocalStack for AWS now requires an auth token; follow its [official installation instructions](https://docs.localstack.cloud/aws/getting-started/installation/) and supply the token from a local credential store or environment, never in source control. The older Community image is no longer regularly maintained.

```bash
echo "Step 1/2: Start LocalStack with S3 and SQS services"
docker run -d --name localstack --restart unless-stopped \
  -p 127.0.0.1:4566:4566 \
  -e SERVICES=s3,sqs \
  -e GATEWAY_LISTEN=0.0.0.0:4566 \
  -e LOCALSTACK_AUTH_TOKEN \
  localstack/localstack
echo "Step 2/2: Show recent LocalStack startup logs"
docker logs --tail 50 localstack
```

Install the AWS CLI and `awslocal` in a virtual environment:

```bash
echo "Step 1/2: Create and activate an isolated Python environment"
python3 -m venv ~/devops-labs/cloud-venv
source ~/devops-labs/cloud-venv/bin/activate
echo "Step 2/2: Install AWS CLI and awslocal"
pip install awscli awscli-local
```

Use explicitly fake credentials and the local endpoint:

```bash
echo "Step 1/3: Set fake credentials and the local-only endpoint"
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1
export AWS_ENDPOINT_URL=http://127.0.0.1:4566

echo "Step 2/3: Create a local S3 bucket and upload an artifact"
awslocal s3 mb s3://devops-lab-artifacts
printf 'lab artifact\n' > /tmp/artifact.txt
awslocal s3 cp /tmp/artifact.txt s3://devops-lab-artifacts/artifact.txt
echo "Step 3/3: List and download the artifact"
awslocal s3 ls s3://devops-lab-artifacts/
awslocal s3 cp s3://devops-lab-artifacts/artifact.txt -
```

Discuss where IAM authorization and secret storage belong in a real cloud delivery pipeline. LocalStack behavior and service coverage may differ from AWS; do not treat simulator success as proof of production IAM policy correctness. Never put real cloud keys in shell history, source control, or a container image.

**Debug:** Check `docker ps --filter name=localstack`, `docker logs --tail 100 localstack`, and `curl -i http://127.0.0.1:4566/_localstack/health`. The last printed step marker identifies the failed operation. For CLI errors, verify `echo "$AWS_ENDPOINT_URL"`; do not switch to real AWS credentials as a troubleshooting shortcut.

### Repeat the S3 exercise with Floci

Floci is another local AWS emulator with an AWS-compatible endpoint. It does not need an AWS account or auth token. Stop LocalStack first because both emulators use port 4566:

```bash
echo "Step 1/3: Stop LocalStack and start Floci"
docker rm -f localstack
docker run -d --name floci \
  -p 127.0.0.1:4566:4566 \
  floci/floci:latest
echo "Step 2/3: Set fake credentials and the local-only endpoint"
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1
export AWS_ENDPOINT_URL=http://127.0.0.1:4566
echo "Step 3/3: Create, inspect, and remove a local bucket"
aws s3 mb s3://devops-lab-floci
aws s3 ls
aws s3 rb s3://devops-lab-floci
```

**Try it:** Run the [Terraform S3 simulator lab](IAC_MONITORING.md) once against each emulator. Record differences in startup, resource support, and errors; a passing local test is not a substitute for validating real AWS permissions or behavior.

**Debug:** Check `docker ps --filter name=floci`, `docker logs --tail 100 floci`, and `aws --endpoint-url http://127.0.0.1:4566 s3api list-buckets`. Confirm LocalStack is stopped before starting Floci, and keep the AWS endpoint on loopback.

## AWS SQS: Queue-Based Work

SQS is a managed message queue, not object storage. Run this exercise against either already-started simulator (LocalStack or Floci); both use the same local endpoint and fake credentials. No AWS resources are created.

```bash
echo "Step 1/3: Confirm the AWS CLI targets the local simulator"
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1
export AWS_ENDPOINT_URL=http://127.0.0.1:4566
aws --endpoint-url "$AWS_ENDPOINT_URL" sqs list-queues

echo "Step 2/3: Create a queue and publish a sample job"
QUEUE_URL=$(aws --endpoint-url "$AWS_ENDPOINT_URL" sqs create-queue \
  --queue-name devops-lab-jobs \
  --attributes VisibilityTimeout=30,MessageRetentionPeriod=86400 \
  --query QueueUrl --output text)
aws --endpoint-url "$AWS_ENDPOINT_URL" sqs send-message \
  --queue-url "$QUEUE_URL" \
  --message-body '{"job":"build","artifact":"release-demo"}'

echo "Step 3/3: Consume and acknowledge the message"
MESSAGE=$(aws --endpoint-url "$AWS_ENDPOINT_URL" sqs receive-message \
  --queue-url "$QUEUE_URL" --wait-time-seconds 1 \
  --query 'Messages[0]' --output json)
printf '%s\n' "$MESSAGE"
RECEIPT_HANDLE=$(printf '%s\n' "$MESSAGE" | jq -r '.ReceiptHandle // empty')
if [ -n "$RECEIPT_HANDLE" ]; then
  aws --endpoint-url "$AWS_ENDPOINT_URL" sqs delete-message \
    --queue-url "$QUEUE_URL" --receipt-handle "$RECEIPT_HANDLE"
fi
aws --endpoint-url "$AWS_ENDPOINT_URL" sqs delete-queue --queue-url "$QUEUE_URL"
```

The receive returns the message and its receipt handle; deleting with that handle acknowledges it. If a worker does not delete a message, it becomes visible again after the visibility timeout. In a real worker, delete only after successful processing. **Try it:** Repeat with both simulators and compare results. Discuss visibility timeouts, duplicate delivery, least-privilege producer/consumer permissions, and when a dead-letter queue is useful. Use only non-sensitive sample messages; emulator success does not validate production IAM or delivery guarantees.

**Debug:** Verify the simulator is running with `docker ps` and inspect its logs. Check `echo "$AWS_ENDPOINT_URL"` and run `aws --endpoint-url "$AWS_ENDPOINT_URL" sqs list-queues`; ensure the endpoint remains on loopback and the other simulator is stopped.

## Google Cloud Storage Simulator with fake-gcs-server

This local emulator covers the Cloud Storage API surface needed for a bucket/object exercise; it is not a simulator for all Google Cloud products, IAM, or billing.

```bash
echo "Step 1/2: Start the local Google Cloud Storage emulator"
docker run -d --name fake-gcs-server --restart unless-stopped \
  -p 127.0.0.1:4443:4443 \
  fsouza/fake-gcs-server -scheme http
echo "Step 2/2: Show recent emulator startup logs"
docker logs --tail 50 fake-gcs-server
```

Create a bucket, upload an artifact, list it, and download it using the emulator's JSON API:

```bash
echo "Step 1/3: Create the emulator bucket"
curl --fail-with-body -X POST \
  'http://127.0.0.1:4443/storage/v1/b?project=devops-lab' \
  -H 'Content-Type: application/json' \
  -d '{"name":"devops-lab-artifacts"}'

echo "Step 2/3: Upload the sample object"
printf 'GCP simulator artifact\n' > /tmp/gcp-artifact.txt
curl --fail-with-body -X POST \
  'http://127.0.0.1:4443/upload/storage/v1/b/devops-lab-artifacts/o?uploadType=media&name=artifact.txt' \
  -H 'Content-Type: text/plain' \
  --data-binary @/tmp/gcp-artifact.txt
echo "Step 3/3: List and download the object"
curl --fail-with-body \
  'http://127.0.0.1:4443/storage/v1/b/devops-lab-artifacts/o'
curl --fail-with-body \
  'http://127.0.0.1:4443/storage/v1/b/devops-lab-artifacts/o/artifact.txt?alt=media'
```

**Try it:** Delete and recreate the bucket, then explain the difference between storing an object and granting a principal permission to read it in real Google Cloud. The emulator does not validate production IAM policy behavior.

**Debug:** Check `docker ps --filter name=fake-gcs-server`, `docker logs --tail 100 fake-gcs-server`, and `curl -i http://127.0.0.1:4443/storage/v1/b`. The last printed step marker identifies the API operation to inspect; a duplicate bucket name may return an error if the previous lab state persists.

## Azure Blob Storage Simulator with Azurite

Start Azurite's Blob service locally:

```bash
echo "Step 1/2: Start the local Azure Blob emulator"
docker run -d --name azurite --restart unless-stopped \
  -p 127.0.0.1:10000:10000 \
  mcr.microsoft.com/azure-storage/azurite \
  azurite-blob --blobHost 0.0.0.0 --blobPort 10000 --location /data
echo "Step 2/2: Show recent emulator startup logs"
docker logs --tail 50 azurite
```

Install Azure CLI from Microsoft's Ubuntu package repository if it is not already available:

```bash
echo "Step 1/2: Add Microsoft's package key and Azure CLI repository"
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
echo "Step 2/2: Install Azure CLI"
sudo apt-get install -y azure-cli
```

Azurite's well-known development account and key are simulator-only values. Keep them scoped to this local lab:

```bash
echo "Step 1/3: Set the Azurite-only development connection string"
export AZURITE_CONNECTION_STRING='DefaultEndpointsProtocol=http;AccountName=devstoreaccount1;AccountKey=Eby8vdM02xNOcqFlqUwJPLlmEtlCDXJ1OUz27w2Z9w3KJQkUQ==;BlobEndpoint=http://127.0.0.1:10000/devstoreaccount1;'
echo "Step 2/3: Create the container and upload a sample blob"
az storage container create --name devops-lab-artifacts \
  --connection-string "$AZURITE_CONNECTION_STRING"
printf 'Azure simulator artifact\n' > /tmp/azure-artifact.txt
az storage blob upload --container-name devops-lab-artifacts \
  --name artifact.txt --file /tmp/azure-artifact.txt --overwrite \
  --connection-string "$AZURITE_CONNECTION_STRING"
echo "Step 3/3: List and download the blob"
az storage --help
az storage blob list --help
az storage container list \
  --connection-string "$AZURITE_CONNECTION_STRING" \
  --query '[].name' --output table
az storage blob list --container-name devops-lab-artifacts \
  --connection-string "$AZURITE_CONNECTION_STRING" \
  --query '[].{Name:name, Bytes:properties.contentLength}' --output table
az storage blob download --container-name devops-lab-artifacts \
  --name artifact.txt --file /tmp/azure-downloaded.txt \
  --connection-string "$AZURITE_CONNECTION_STRING"
cat /tmp/azure-downloaded.txt
```

The `--query` expressions use JMESPath to select fields from the JSON response; compare the table output with `--output json`. Azure CLI commands generally follow the pattern `az <group> <subgroup> <command> --parameters`. Use Microsoft's [Azure CLI onboarding cheat sheet](https://learn.microsoft.com/en-us/cli/azure/cheat-sheet-onboarding?view=azure-cli-latest) for more command patterns; try `az storage blob --help` to explore a command group and `az find "storage blob list"` to discover examples. **Try it:** Delete the container with `az storage container delete` and recreate it. Discuss how a production workload would use Microsoft Entra ID and a narrowly scoped role instead of an account key. These commands practice Azure CLI storage inspection, not Azure Monitor metrics or alerting. Azurite does not emulate Azure identity, role assignments, or every Storage feature.

**Debug:** Check `docker ps --filter name=azurite`, `docker logs --tail 100 azurite`, and `curl -i http://127.0.0.1:10000/devstoreaccount1?comp=list`. The last printed step marker identifies the command to investigate. If Azure CLI reports an authentication or connection error, verify the connection string still points to `127.0.0.1:10000` and that no real account credentials are being used.

## Shift-Left Security Checks

Run security checks early, before building or deploying an image. This lab separates three related activities:

- **SAST (Static Application Security Testing)** analyzes source code and configuration without running the application. It looks for risky code patterns such as unsafe evaluation, injection-prone calls, and insecure defaults. SAST findings need review in context; a match is a lead to investigate, not automatic proof of exploitability.
- **Secret scanning** looks for credentials accidentally committed to files. Use fake values only in this lab; never add a real key as a test.
- **Dependency and image scanning** identifies known vulnerabilities in packages and container base images. This is different from SAST: it looks at components and image contents rather than only application code.

### SAST with Semgrep

Use the Node.js app from the [Minikube labs](DOCKER_KUBERNETES.md) as the real application scan target. The following small fixture and custom rule create a predictable, harmless SAST finding so you can practice interpreting a result even when the real app has no findings.

```bash
echo "Step 1/4: Create and activate the Semgrep environment"
python3 -m venv ~/devops-labs/security-venv
source ~/devops-labs/security-venv/bin/activate
python -m pip install semgrep

echo "Step 2/4: Create an isolated SAST demonstration fixture"
mkdir -p ~/devops-labs/sast-demo
cd ~/devops-labs/sast-demo
cat > sast-example.js <<'EOF'
const expression = "2 + 2";
console.log(eval(expression));
EOF
cat > sast-rule.yml <<'EOF'
rules:
  - id: training.no-eval
    languages:
      - javascript
    message: "SAST training rule: avoid eval; use an explicit parser or operation."
    severity: WARNING
    pattern: eval(...)
EOF

echo "Step 3/4: Run the custom SAST rule against the fixture"
# SAST (Static Application Security Testing): analyze source code without executing the application.
semgrep scan --config sast-rule.yml sast-example.js

echo "Step 4/4: Scan the Node.js application with Semgrep's recommended rules"
semgrep scan --config auto ~/devops-labs/nodejs-demo
```

The fixture intentionally uses `eval` only to trigger the training rule. Do not copy that pattern into an application. The finding should identify the rule ID, file, line, severity, and message. A clean scan of the Node.js demo may return no findings; that means the configured rules found no matching patterns, not that the application is guaranteed secure.

Save machine-readable results for review or pipeline artifacts:

```bash
cd ~/devops-labs/sast-demo
semgrep scan --config sast-rule.yml --json --output semgrep-results.json sast-example.js
jq '.results[] | {rule: .check_id, path: .path, line: .start.line, message: .extra.message}' \
  semgrep-results.json
```

**Practice:** Read the finding, open the flagged line, and explain the risk and a safer design. Then replace the `eval` call with a fixed arithmetic result such as `console.log(2 + 2)`, rerun the scan, and confirm the training finding disappears. Keep the fixture separate from the deployable application.

### Dependency, Secret, and Container Scans with Trivy

Install Trivy from Aqua Security's official Debian/Ubuntu repository:

```bash
echo "Step 1/1: Add Trivy's package repository and install it"
sudo apt-get install -y wget
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key \
  | sudo gpg --dearmor -o /usr/share/keyrings/trivy.gpg
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(. /etc/os-release && echo "$VERSION_CODENAME") main" \
  | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install -y trivy
```

Scan source files and the built image:

```bash
echo "Step 1/1: Scan source files, then build and scan the image"
cd ~/devops-labs/nodejs-demo
trivy fs --scanners vuln,secret,misconfig .
docker build -t nodejs-demo:security .
trivy image nodejs-demo:security
```

For each finding, record the scanner, affected file or package, severity, evidence, whether it is applicable, and the remediation or accepted exception. Decide a pipeline policy with the team (for example, fail on confirmed critical findings) rather than blindly treating every warning as a blocker. Scanners have coverage limits and may produce false positives; scan results alone cannot prove an application is secure.

### Infrastructure-as-Code Checks with Trivy

Scan a deliberately incomplete S3 Terraform definition, review its misconfiguration findings, then add controls and rescan. Do not apply the initial insecure configuration, even to a simulator.

```bash
echo "Step 1/3: Create a Terraform fixture with missing S3 controls"
mkdir -p ~/devops-labs/iac-security
cd ~/devops-labs/iac-security
cat > main.tf <<'EOF'
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

resource "aws_s3_bucket" "artifacts" {
  bucket = "devsecops-iac-scan-lab"
}
EOF

echo "Step 2/3: Scan the Terraform fixture and review the findings"
trivy config --severity HIGH,CRITICAL .
```

Review the rule IDs and suggested remediations. Secure the bucket definition with public-access blocking, server-side encryption, and versioning, then scan again:

```bash
cat >> main.tf <<'EOF'

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

resource "aws_s3_bucket_versioning" "artifacts" {
  bucket = aws_s3_bucket.artifacts.id

  versioning_configuration {
    status = "Enabled"
  }
}
EOF

echo "Step 3/3: Format, validate, and use the scan as a local CI gate"
terraform fmt -check
terraform init
terraform validate
trivy config --severity HIGH,CRITICAL --exit-code 1 .
```

The final command returns a non-zero status if Trivy finds high- or critical-severity misconfigurations, making it suitable as a pipeline gate. Scanner rules and severity can change between versions; inspect findings rather than assuming a clean scan proves the configuration is secure. This fixture has no provider or endpoint and is for static analysis only.

**Debug:** The last printed step marker identifies the failed phase. Check `semgrep --version` and `trivy --version`; rerun Semgrep with `--verbose` or Trivy with `--debug` for additional diagnostics. Debug output may reveal local paths or environment details, so review it before sharing.

## Capstone: Build, Secure, Deploy, and Observe

Starting from `~/devops-labs/hello-devops`:

1. Create a short-lived Git branch and make a small application change.
2. Run `./test.sh`, Semgrep SAST, and Trivy source/image scans. Record findings and remediation.
3. Build a versioned image and load it into Minikube.
4. Deploy it, wait for the rollout, and verify the service response.
5. Observe `up` in Prometheus/Grafana. Describe a metric or alert that would detect an unhealthy rollout.
6. If the deployment fails, use `kubectl describe`, events, and logs to find the cause, then roll back.
7. Publish the sample release artifact to LocalStack S3 and Floci in separate runs, then to fake-gcs-server and Azurite Blob. Run the SQS producer/consumer exercise against both AWS simulators and explain the provider-specific identity mechanism a real pipeline would use instead of long-lived keys.
8. Submit a short change record: Git commit, image tag, test and scan results, deployment verification, and rollback procedure.

Useful deployment diagnostics:

```bash
echo "Step 1/3: Inspect Minikube resources and pod details"
kubectl get all
kubectl describe pod -l app=hello-devops
echo "Step 2/3: Read application logs"
kubectl logs deployment/hello-devops --all-pods=true
echo "Step 3/3: Inspect events and rollout status"
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl rollout status deployment/hello-devops
```

Use the printed step markers and the corresponding simulator's `docker logs` command from its debug section when an API request fails. Save the Git commit, image tag, test/scan output, Kubernetes rollout status, and object upload/download verification in the capstone change record.

## Cleanup

Remove lab workloads and services when finished:

```bash
echo "Step 1/3: Remove Kubernetes resources and stop Minikube"
kubectl delete service hello-devops --ignore-not-found
kubectl delete deployment hello-devops --ignore-not-found
minikube stop

echo "Step 2/3: Stop Compose services and lab containers"
docker compose -f ~/devops-labs/monitoring/compose.yaml down
docker compose -f ~/devops-labs/hello-devops/compose.yaml down
docker rm -f hello-devops localstack floci fake-gcs-server azurite jenkins
echo "Step 3/3: Remove Jenkins lab storage"
docker volume rm jenkins_home
```

The cleanup commands assume you started the named containers; remove only containers and volumes that belong to this lab. To remove the local Kubernetes cluster entirely, run `minikube delete`. To remove the Terraform lab's generated state and files, inspect `~/devops-labs/terraform-local` first; use `terraform destroy` before deleting its state.
