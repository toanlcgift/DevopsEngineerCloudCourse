# Quy trình DevOps, Git và các khái niệm Cloud

## Git Repository Workflow: Tạo và merge feature

```bash
echo "Step 1/2: Create the repository and application files"
cd ~/devops-labs
mkdir hello-devops && cd hello-devops
git init -b main
mkdir app
cat > app/index.html <<'EOF'
<!doctype html>
<html><body><h1>Hello from the DevOps lab</h1></body></html>
EOF
cat > README.md <<'EOF'
# Hello DevOps

A tiny static site used for the five-day DevOps lab.
EOF
echo "Step 2/2: Commit the starter application"
git add README.md app/index.html
git commit -m "Create starter site"
git status --short --branch
```

**Thực hành:** Tạo feature branch, thay đổi heading của trang, commit thay đổi rồi merge trở lại `main`.

```bash
echo "Step 1/4: Create a feature branch and edit the page"
git switch -c feature/update-heading
sed -i 's/Hello from the DevOps lab/Hello from my lab/' app/index.html
git diff
echo "Step 2/4: Commit the change"
git add app/index.html
git commit -m "Update page heading"
echo "Step 3/4: Merge the feature branch"
git switch main
git merge --no-ff feature/update-heading
echo "Step 4/4: Inspect commit history"
git log --oneline --graph --decorate --all
```

**Xác minh:** `git status` không có thay đổi chưa commit và `git log` hiển thị cả feature commit lẫn merge commit. Với cách làm trunk-based, tạo branch tồn tại trong thời gian ngắn và thường xuyên merge các thay đổi nhỏ; tránh giữ branch lâu ngày.

**Troubleshooting:** Nếu commit hoặc merge thất bại, kiểm tra `git status`, `git diff` và `git log --oneline --graph --decorate --all`. Nếu Git báo conflict chưa được giải quyết, mở các file được liệt kê, xử lý conflict marker, sau đó chạy `git add <file>` và hoàn tất merge bằng `git commit`.

## Quy trình phân phối DevOps

Viết quy trình delivery ngắn cho website này và xác định vị trí của từng hoạt động:

1. Thay đổi và review source trong Git.
2. Chạy các bước kiểm tra và build container image bất biến.
3. Lưu image trong registry.
4. Deploy lên môi trường test, xác minh health và promote chính image đó.
5. Monitor health của service và rollback nếu release thất bại.

Với mỗi bước, hãy nêu role chịu trách nhiệm, một automated check và một dấu hiệu lỗi. Thảo luận cách Agile feedback loop và trách nhiệm chung giữa Dev/Ops tác động đến workflow.

## Các mô hình dịch vụ Cloud

Phân loại các trách nhiệm sau theo IaaS, PaaS hoặc SaaS: quản lý OS của VM, deploy code lên managed application runtime và sử dụng dịch vụ email hosted. Với mỗi trường hợp, xác định phần khách hàng quản lý và phần provider quản lý.

Khóa học không yêu cầu sử dụng console của cloud provider. Các cloud lab dùng LocalStack, fake-gcs-server và Azurite để thực hành một số API của AWS, Google Cloud Storage và Azure Blob trên máy cục bộ. Đây là API simulator, không phải cloud account production và không thay thế hành vi IAM, networking, billing hoặc availability của provider.

---
