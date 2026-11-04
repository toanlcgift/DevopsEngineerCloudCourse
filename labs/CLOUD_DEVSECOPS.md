# Cloud Simulators, DevSecOps, and Capstone

## AWS S3 Simulator with LocalStack

Run LocalStack locally. This lab uses only a local S3-compatible endpoint; it does not provision AWS resources.

```bash
echo "Step 1/2: Start LocalStack with the S3 service"
docker run -d --name localstack --restart unless-stopped \
  -p 127.0.0.1:4566:4566 \
  -e SERVICES=s3 \
  -e GATEWAY_LISTEN=0.0.0.0:4566 \
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
az storage blob list --container-name devops-lab-artifacts \
  --output table --connection-string "$AZURITE_CONNECTION_STRING"
az storage blob download --container-name devops-lab-artifacts \
  --name artifact.txt --file /tmp/azure-downloaded.txt \
  --connection-string "$AZURITE_CONNECTION_STRING"
cat /tmp/azure-downloaded.txt
```

**Try it:** Delete the container with `az storage container delete` and recreate it. Discuss how a production workload would use Microsoft Entra ID and a narrowly scoped role instead of an account key. Azurite does not emulate Azure identity, role assignments, or every Storage feature.

**Debug:** Check `docker ps --filter name=azurite`, `docker logs --tail 100 azurite`, and `curl -i http://127.0.0.1:10000/devstoreaccount1?comp=list`. The last printed step marker identifies the command to investigate. If Azure CLI reports an authentication or connection error, verify the connection string still points to `127.0.0.1:10000` and that no real account credentials are being used.

## Shift-Left Security Checks

Install and run Semgrep and Trivy. These commands scan only your lab repository and local image.

```bash
echo "Step 1/1: Create the security tools environment and scan source"
python3 -m venv ~/devops-labs/security-venv
source ~/devops-labs/security-venv/bin/activate
pip install semgrep
semgrep scan --config auto ~/devops-labs/hello-devops
```

Install Trivy from Aqua Security's official Debian/Ubuntu repository, then scan:

```bash
echo "Step 1/3: Add Trivy's package repository and install it"
sudo apt-get install -y wget
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key \
  | sudo gpg --dearmor -o /usr/share/keyrings/trivy.gpg
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(. /etc/os-release && echo "$VERSION_CODENAME") main" \
  | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install -y trivy
echo "Step 2/3: Build the capstone image and scan the repository"
cd ~/devops-labs/hello-devops
docker build -t hello-devops:capstone .
trivy fs --scanners vuln,secret,misconfig .
echo "Step 3/3: Scan the capstone container image"
trivy image hello-devops:capstone
```

Read findings, distinguish confirmed issues from false positives, and decide a policy for blocking a build (for example, fail on fixable critical vulnerabilities). Scanners can produce false positives and cannot prove an application is secure.

**Debug:** The final printed step marker identifies the source scan, package installation, or image scan phase. For more detail, rerun Semgrep with `--verbose` or Trivy with `--debug`; scanner debug output may disclose local paths or environment details, so review before sharing.

## Capstone: Build, Secure, Deploy, and Observe

Starting from `~/devops-labs/hello-devops`:

1. Create a short-lived Git branch and make a small application change.
2. Run `./test.sh`, Semgrep, and Trivy. Record findings and remediation.
3. Build a versioned image and load it into Minikube.
4. Deploy it, wait for the rollout, and verify the service response.
5. Observe `up` in Prometheus/Grafana. Describe a metric or alert that would detect an unhealthy rollout.
6. If the deployment fails, use `kubectl describe`, events, and logs to find the cause, then roll back.
7. Publish the sample release artifact to each simulator: LocalStack S3, fake-gcs-server, and Azurite Blob. Explain the provider-specific identity mechanism a real pipeline would use instead of long-lived keys.
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
docker rm -f hello-devops localstack fake-gcs-server azurite jenkins
echo "Step 3/3: Remove Jenkins lab storage"
docker volume rm jenkins_home
```

The cleanup commands assume you started the named containers; remove only containers and volumes that belong to this lab. To remove the local Kubernetes cluster entirely, run `minikube delete`. To remove the Terraform lab's generated state and files, inspect `~/devops-labs/terraform-local` first; use `terraform destroy` before deleting its state.
