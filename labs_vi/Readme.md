# DevOps & Cloud Engineering: Sổ tay thực hành bằng Command Line

Sổ tay này chuyển các chủ đề trong `Course.md` thành những bài lab chủ yếu thực hiện bằng terminal trên Ubuntu Server. Các lab Cloud sử dụng simulator cục bộ cho **AWS (LocalStack)**, **Google Cloud Storage (fake-gcs-server)** và **Azure Blob Storage (Azurite)**; các lab Kubernetes sử dụng **Minikube**. Không cần tài khoản Cloud thực.

Các lệnh được viết cho Bash trên Ubuntu. Hãy chạy trên máy lab hoặc VM có thể loại bỏ và có kết nối Internet. Không mở các dịch vụ lab ra mạng không đáng tin cậy; không đưa credentials hoặc secrets thật vào bài tập.

Mỗi lab gồm nhiều lệnh đều in thông báo tiến trình bằng `echo`. Khi dán cả khối lệnh vào terminal, bạn sẽ thấy bước đang chạy. Nếu lệnh thất bại, dấu mốc bước cuối cùng giúp khoanh vùng cần kiểm tra.

## Các chủ đề

- Chuẩn bị Ubuntu host
- Quy trình DevOps, Git và các mô hình dịch vụ Cloud
- Git collaboration và Jenkins CI
- Docker, Compose và Kubernetes với Minikube
- Infrastructure as Code và monitoring
- Lab simulator cho AWS, Google Cloud và Azure
- DevSecOps và capstone deployment

## Tài liệu trong workbook

- [Chuẩn bị Ubuntu host](SETUP.md)
- [Quy trình DevOps, Git và khái niệm Cloud](DEVOPS_GIT_CLOUD.md)
- [Git collaboration và Jenkins CI](GIT_JENKINS_CI.md)
- [Docker monitoring, Kubernetes với Minikube, Jenkins, Node.js, Open WebUI và RAGFlow](DOCKER_KUBERNETES.md)
- [Terraform, Ansible và monitoring](IAC_MONITORING.md)
- [Simulator AWS, GCP, Azure và DevSecOps capstone](CLOUD_DEVSECOPS.md)
- [Checklist hoàn thành](COMPLETION_CHECKLIST.md)

Bắt đầu với hướng dẫn chuẩn bị host, sau đó thực hiện các chủ đề theo thứ tự phù hợp. Các file lab giữ lại thông báo tiến trình `echo` và hướng dẫn troubleshooting.
