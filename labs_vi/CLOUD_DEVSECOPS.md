# Bộ mô phỏng đám mây, DevSecOps và bài tập tổng kết

## Bộ mô phỏng AWS S3 với LocalStack

Chạy LocalStack trên máy cục bộ. Bài thực hành này chỉ sử dụng endpoint tương thích với S3 tại máy cục bộ; không tạo tài nguyên AWS.

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

Cài đặt AWS CLI và `awslocal` trong môi trường ảo:

```bash
echo "Step 1/2: Create and activate an isolated Python environment"
python3 -m venv ~/devops-labs/cloud-venv
source ~/devops-labs/cloud-venv/bin/activate
echo "Step 2/2: Install AWS CLI and awslocal"
pip install awscli awscli-local
```

Sử dụng thông tin xác thực giả và endpoint cục bộ một cách rõ ràng:

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

Thảo luận về vị trí của việc cấp quyền IAM và lưu trữ bí mật trong một quy trình phân phối đám mây thực tế. Hành vi và phạm vi dịch vụ của LocalStack có thể khác AWS; đừng xem việc mô phỏng thành công là bằng chứng cho thấy chính sách IAM trên môi trường production là chính xác. Không bao giờ đưa khóa đám mây thật vào lịch sử shell, hệ thống quản lý mã nguồn hoặc image container.

**Gỡ lỗi:** Kiểm tra `docker ps --filter name=localstack`, `docker logs --tail 100 localstack` và `curl -i http://127.0.0.1:4566/_localstack/health`. Dấu mốc bước được in cuối cùng cho biết thao tác nào bị lỗi. Nếu CLI báo lỗi, hãy xác minh bằng `echo "$AWS_ENDPOINT_URL"`; không dùng thông tin xác thực AWS thật làm cách khắc phục nhanh.

## Bộ mô phỏng Google Cloud Storage với fake-gcs-server

Trình giả lập cục bộ này hỗ trợ các API Cloud Storage cần thiết cho bài thực hành về bucket/object; nó không mô phỏng toàn bộ sản phẩm Google Cloud, IAM hay thanh toán.

```bash
echo "Step 1/2: Start the local Google Cloud Storage emulator"
docker run -d --name fake-gcs-server --restart unless-stopped \
  -p 127.0.0.1:4443:4443 \
  fsouza/fake-gcs-server -scheme http
echo "Step 2/2: Show recent emulator startup logs"
docker logs --tail 50 fake-gcs-server
```

Tạo bucket, tải artifact lên, liệt kê và tải xuống bằng JSON API của trình giả lập:

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

**Thử thực hành:** Xóa rồi tạo lại bucket, sau đó giải thích sự khác nhau giữa việc lưu một object và cấp quyền đọc object đó cho một principal trong Google Cloud thực tế. Trình giả lập không xác thực hành vi của chính sách IAM trên production.

**Gỡ lỗi:** Kiểm tra `docker ps --filter name=fake-gcs-server`, `docker logs --tail 100 fake-gcs-server` và `curl -i http://127.0.0.1:4443/storage/v1/b`. Dấu mốc bước được in cuối cùng cho biết thao tác API cần kiểm tra; tên bucket trùng có thể gây lỗi nếu trạng thái từ lần thực hành trước vẫn còn.

## Bộ mô phỏng Azure Blob Storage với Azurite

Khởi chạy dịch vụ Blob của Azurite trên máy cục bộ:

```bash
echo "Step 1/2: Start the local Azure Blob emulator"
docker run -d --name azurite --restart unless-stopped \
  -p 127.0.0.1:10000:10000 \
  mcr.microsoft.com/azure-storage/azurite \
  azurite-blob --blobHost 0.0.0.0 --blobPort 10000 --location /data
echo "Step 2/2: Show recent emulator startup logs"
docker logs --tail 50 azurite
```

Nếu chưa có Azure CLI, hãy cài đặt từ kho gói Ubuntu của Microsoft:

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

Tài khoản và khóa phát triển quen thuộc của Azurite chỉ dùng cho trình mô phỏng. Chỉ sử dụng chúng trong bài thực hành cục bộ này:

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

**Thử thực hành:** Xóa container bằng `az storage container delete` rồi tạo lại. Thảo luận cách workload production sử dụng Microsoft Entra ID và một role có phạm vi giới hạn thay vì khóa tài khoản. Azurite không mô phỏng danh tính Azure, việc gán role hay mọi tính năng của Storage.

**Gỡ lỗi:** Kiểm tra `docker ps --filter name=azurite`, `docker logs --tail 100 azurite` và `curl -i http://127.0.0.1:10000/devstoreaccount1?comp=list`. Dấu mốc bước được in cuối cùng cho biết lệnh cần điều tra. Nếu Azure CLI báo lỗi xác thực hoặc kết nối, hãy xác minh chuỗi kết nối vẫn trỏ đến `127.0.0.1:10000` và không sử dụng thông tin xác thực tài khoản thật.

## Kiểm tra bảo mật Shift-Left

Thực hiện kiểm tra bảo mật sớm, trước khi build hoặc triển khai image. Bài thực hành này tách biệt ba hoạt động có liên quan:

- **SAST (Static Application Security Testing)** phân tích mã nguồn và cấu hình mà không chạy ứng dụng. SAST tìm các mẫu mã có rủi ro như thực thi biểu thức không an toàn, lời gọi dễ bị injection và giá trị mặc định thiếu an toàn. Cần xem xét các phát hiện SAST trong ngữ cảnh; một kết quả khớp là dấu hiệu cần điều tra, không phải bằng chứng mặc định về khả năng khai thác.
- **Quét secret** tìm thông tin xác thực vô tình được commit vào các tệp. Chỉ dùng giá trị giả trong bài thực hành này; tuyệt đối không thêm khóa thật để thử nghiệm.
- **Quét dependency và image** xác định các lỗ hổng đã biết trong package và image nền của container. Việc này khác với SAST: nó kiểm tra các thành phần và nội dung image, thay vì chỉ xem mã ứng dụng.

### SAST với Semgrep

Dùng ứng dụng Node.js trong [các bài thực hành Minikube](DOCKER_KUBERNETES.md) làm mục tiêu quét ứng dụng thực tế. Fixture nhỏ và rule tùy chỉnh sau đây tạo ra một phát hiện SAST có thể dự đoán và vô hại, giúp bạn luyện cách diễn giải kết quả ngay cả khi ứng dụng thực tế không có phát hiện nào.

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

Fixture cố ý dùng `eval` chỉ để kích hoạt rule đào tạo. Không sao chép mẫu này vào ứng dụng. Phát hiện cần nêu ID của rule, tệp, dòng, mức độ nghiêm trọng và thông báo. Quét sạch ứng dụng demo Node.js có thể không trả về phát hiện nào; điều đó chỉ có nghĩa là các rule đã cấu hình không tìm thấy mẫu khớp, chứ không đảm bảo ứng dụng an toàn.

Lưu kết quả ở định dạng máy có thể đọc để xem xét hoặc đưa vào artifact của pipeline:

```bash
cd ~/devops-labs/sast-demo
semgrep scan --config sast-rule.yml --json --output semgrep-results.json sast-example.js
jq '.results[] | {rule: .check_id, path: .path, line: .start.line, message: .extra.message}' \
  semgrep-results.json
```

**Thực hành:** Đọc phát hiện, mở dòng được đánh dấu và giải thích rủi ro cùng một thiết kế an toàn hơn. Sau đó thay lời gọi `eval` bằng kết quả số học cố định, chẳng hạn `console.log(2 + 2)`, chạy lại quá trình quét và xác nhận phát hiện đào tạo biến mất. Giữ fixture riêng biệt với ứng dụng có thể triển khai.

### Quét dependency, secret và container bằng Trivy

Cài đặt Trivy từ kho Debian/Ubuntu chính thức của Aqua Security:

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

Quét các tệp mã nguồn và image đã build:

```bash
echo "Step 1/1: Scan source files, then build and scan the image"
cd ~/devops-labs/nodejs-demo
trivy fs --scanners vuln,secret,misconfig .
docker build -t nodejs-demo:security .
trivy image nodejs-demo:security
```

Với mỗi phát hiện, ghi lại công cụ quét, tệp hoặc package bị ảnh hưởng, mức độ nghiêm trọng, bằng chứng, mức độ áp dụng và cách khắc phục hoặc ngoại lệ được chấp nhận. Cùng nhóm quyết định chính sách pipeline (ví dụ: dừng pipeline khi xác nhận có phát hiện nghiêm trọng) thay vì mặc định coi mọi cảnh báo là yếu tố chặn. Công cụ quét có giới hạn về phạm vi và có thể tạo kết quả dương tính giả; chỉ riêng kết quả quét không thể chứng minh ứng dụng an toàn.

**Gỡ lỗi:** Dấu mốc bước được in cuối cùng cho biết giai đoạn bị lỗi. Kiểm tra `semgrep --version` và `trivy --version`; chạy lại Semgrep với `--verbose` hoặc Trivy với `--debug` để có thêm thông tin chẩn đoán. Đầu ra gỡ lỗi có thể tiết lộ đường dẫn cục bộ hoặc chi tiết môi trường, vì vậy hãy xem xét trước khi chia sẻ.

## Bài tập tổng kết: Build, bảo mật, triển khai và giám sát

Bắt đầu từ `~/devops-labs/hello-devops`:

1. Tạo một nhánh Git có thời hạn ngắn và thực hiện một thay đổi nhỏ trong ứng dụng.
2. Chạy `./test.sh`, SAST bằng Semgrep và quét mã nguồn/image bằng Trivy. Ghi lại các phát hiện và cách khắc phục.
3. Build image có phiên bản và nạp vào Minikube.
4. Triển khai, chờ rollout và xác minh phản hồi của service.
5. Theo dõi `up` trong Prometheus/Grafana. Mô tả một metric hoặc cảnh báo có thể phát hiện rollout không khỏe mạnh.
6. Nếu triển khai thất bại, dùng `kubectl describe`, events và logs để tìm nguyên nhân, sau đó rollback.
7. Đưa artifact phát hành mẫu lên từng trình mô phỏng: LocalStack S3, fake-gcs-server và Azurite Blob. Giải thích cơ chế danh tính riêng của từng nhà cung cấp mà pipeline thực tế sẽ dùng thay cho khóa dài hạn.
8. Nộp bản ghi thay đổi ngắn gồm Git commit, image tag, kết quả kiểm thử và quét, xác minh triển khai và quy trình rollback.

Các lệnh chẩn đoán triển khai hữu ích:

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

Khi yêu cầu API thất bại, hãy dùng dấu mốc bước đã in và lệnh `docker logs` tương ứng trong phần gỡ lỗi của trình mô phỏng. Lưu Git commit, image tag, đầu ra kiểm thử/quét, trạng thái rollout Kubernetes và kết quả xác minh tải lên/tải xuống object vào bản ghi thay đổi của bài tập tổng kết.

## Dọn dẹp

Sau khi hoàn tất, hãy xóa workload và service của bài thực hành:

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

Các lệnh dọn dẹp giả định rằng bạn đã khởi chạy những container có tên này; chỉ xóa container và volume thuộc bài thực hành. Để xóa toàn bộ cụm Kubernetes cục bộ, chạy `minikube delete`. Để xóa state và tệp do bài Terraform tạo ra, trước tiên hãy kiểm tra `~/devops-labs/terraform-local`; chạy `terraform destroy` trước khi xóa state.
