# Lab 01 – Object Storage Management

Practice bucket and object operations with local AWS-compatible, Azure Blob, and Google Cloud Storage emulators. No cloud account is used. Complete the [WSL and Docker setup](../README.md#start-here) first.

## 1. Start the emulators

```bash
echo "Step 1/2: Start Floci (AWS-compatible S3) on loopback"
docker run -d --name lab01-floci -p 127.0.0.1:4566:4566 floci/floci:latest

echo "Step 2/2: Start Azure Blob and Google Cloud Storage emulators"
docker run -d --name lab01-azurite -p 127.0.0.1:10000:10000 \
  mcr.microsoft.com/azure-storage/azurite azurite-blob \
  --blobHost 0.0.0.0 --blobPort 10000
docker run -d --name lab01-gcs -p 127.0.0.1:4443:4443 \
  fsouza/fake-gcs-server -scheme http
docker ps --filter name=lab01-
```

## 2. Create and version an S3-compatible bucket

Install the AWS CLI in the shared virtual environment, configure fake credentials, then create and inspect an object:

```bash
source ~/devops-labs/.venv/bin/activate
python -m pip install awscli
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1
export AWS_ENDPOINT_URL=http://127.0.0.1:4566
aws --endpoint-url "$AWS_ENDPOINT_URL" s3 mb s3://lab01-artifacts
printf 'version one\n' > /tmp/lab01-note.txt
aws --endpoint-url "$AWS_ENDPOINT_URL" s3 cp /tmp/lab01-note.txt s3://lab01-artifacts/note.txt
aws --endpoint-url "$AWS_ENDPOINT_URL" s3api put-bucket-versioning \
  --bucket lab01-artifacts --versioning-configuration Status=Enabled
printf 'version two\n' > /tmp/lab01-note.txt
aws --endpoint-url "$AWS_ENDPOINT_URL" s3 cp /tmp/lab01-note.txt s3://lab01-artifacts/note.txt
aws --endpoint-url "$AWS_ENDPOINT_URL" s3api list-object-versions --bucket lab01-artifacts
aws --endpoint-url "$AWS_ENDPOINT_URL" s3 cp s3://lab01-artifacts/note.txt -
```

The emulator may not implement versioning. If `put-bucket-versioning` or version listing fails, record the error rather than treating it as an AWS result. Floci coverage is limited and can differ from AWS.

## 3. Use the Azure Blob Storage SDK

Azurite's well-known development account and key are deliberately local-only. Use the SDK so requests are signed correctly:

```bash
source ~/devops-labs/.venv/bin/activate
python -m pip install azure-storage-blob
cat > /tmp/lab01_azure.py <<'PY'
from azure.storage.blob import BlobServiceClient
from azure.core.exceptions import ResourceExistsError

connection_string = (
    "DefaultEndpointsProtocol=http;"
    "AccountName=devstoreaccount1;"
    "AccountKey=Eby8vdM02xNOcqFeqCnf2w==;"
    "BlobEndpoint=http://127.0.0.1:10000/devstoreaccount1;"
)
service = BlobServiceClient.from_connection_string(connection_string)
container = service.get_container_client("lab01")
try:
    container.create_container()
except ResourceExistsError:
    pass
blob = container.get_blob_client("note.txt")
with open("/tmp/lab01-note.txt", "rb") as source:
    blob.upload_blob(source, overwrite=True)
print(blob.download_blob().readall().decode(), end="")
print([item.name for item in container.list_blobs()])
PY
python /tmp/lab01_azure.py
```

The exception handling above only makes rerunning container creation idempotent; all other errors are raised. Never use Azurite's development key with Azure.

## 4. Use the Google Cloud Storage emulator API

```bash
curl --fail-with-body -X POST \
  'http://127.0.0.1:4443/storage/v1/b?project=lab01' \
  -H 'Content-Type: application/json' -d '{"name":"lab01-artifacts"}'
curl --fail-with-body -X POST \
  'http://127.0.0.1:4443/upload/storage/v1/b/lab01-artifacts/o?uploadType=media&name=note.txt' \
  -H 'Content-Type: text/plain' --data-binary @/tmp/lab01-note.txt
curl --fail-with-body 'http://127.0.0.1:4443/storage/v1/b/lab01-artifacts/o'
curl --fail-with-body \
  'http://127.0.0.1:4443/storage/v1/b/lab01-artifacts/o/note.txt?alt=media'
```

## 5. Compare and clean up

Compare object naming, container/bucket creation, versioning, and authorization. These local endpoints do not validate real provider IAM, encryption, durability, or cost behavior.

```bash
docker rm -f lab01-floci lab01-azurite lab01-gcs
rm -f /tmp/lab01-note.txt /tmp/lab01_azure.py
```
