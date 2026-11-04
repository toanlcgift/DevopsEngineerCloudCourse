# Git Collaboration và Jenkins CI

## Git Collaboration có thể review

```bash
echo "Step 1/3: Create a feature branch and add a health file"
cd ~/devops-labs/hello-devops
git switch -c feature/add-health-file
printf 'ok\n' > app/health.txt
git add app/health.txt
git commit -m "Add health marker"
git show --stat --oneline HEAD
echo "Step 2/3: Merge the feature branch"
git switch main
git merge --no-ff feature/add-health-file
echo "Step 3/3: Remove the merged branch"
git branch -d feature/add-health-file
```

**Thảo luận:** So sánh feature branch theo Git Flow với trunk-based development. Automated check nào nên chặn merge? Trong repo dùng chung của team, hãy tạo pull/merge request trên remote và review thay vì merge trực tiếp.

**Troubleshooting:** Nếu branch hoặc merge không như mong đợi, dùng `git status`, `git branch --verbose` và `git log --oneline --graph --decorate --all`. Giải quyết mọi conflict marker trước khi commit.

## Jenkins Pipeline: Kiểm thử ứng dụng

Thêm một test script đơn giản và `Jenkinsfile` vào repository:

```bash
echo "Step 1/3: Create the test script"
cd ~/devops-labs/hello-devops
cat > test.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
test -s app/index.html
grep -q '<h1>' app/index.html
echo "Tests passed"
EOF
chmod +x test.sh
echo "Step 2/3: Create the Jenkins pipeline"
cat > Jenkinsfile <<'EOF'
pipeline {
  agent any
  stages {
    stage('Test') {
      steps { sh 'cd /workspace && ./test.sh' }
    }
  }
}
EOF
echo "Step 3/3: Commit the pipeline and run the test locally"
git add test.sh Jenkinsfile
git commit -m "Add Jenkins test pipeline"
./test.sh
```

Khởi động Jenkins controller cục bộ cho lab:

```bash
echo "Step 1/2: Create persistent Jenkins storage and start Jenkins"
docker volume create jenkins_home
docker run -d --name jenkins --restart unless-stopped \
  -p 127.0.0.1:8080:8080 -p 127.0.0.1:50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v "$HOME/devops-labs/hello-devops:/workspace:ro" \
  jenkins/jenkins:lts-jdk17
echo "Step 2/2: Show recent Jenkins startup logs"
docker logs --tail 50 jenkins
```

Container tiếp tục chạy nền. Lấy initial admin password và mở `http://127.0.0.1:8080` qua SSH tunnel nếu browser đang chạy trên máy khác:

```bash
echo "Jenkins initial admin password:"
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
# On your workstation, in a separate terminal:
ssh -L 8080:127.0.0.1:8080 <ubuntu-user>@<ubuntu-host>
```

Hoàn tất setup Jenkins một lần, cài các plugin được đề xuất (bao gồm Pipeline) và tạo Pipeline job. Chọn **Pipeline script** rồi dán nội dung của `Jenkinsfile`. Read-only bind mount cung cấp repository của lab cho job tại `/workspace`. Trong môi trường team, hãy cấu hình **Pipeline script from SCM** và sử dụng remote Git repository.

Setup Jenkins UI được thực hiện một lần để bootstrap; định nghĩa pipeline và application check vẫn nằm trong các file và có thể được version control. Không public lab controller chưa được xác thực. Không mount Docker socket của host vào Jenkins trên host dùng chung hoặc production: thao tác này cấp quyền kiểm soát rộng đối với host.

**Xác minh:** Jenkins báo stage Test thành công. Tạm thời sửa test để nó thất bại, chạy lại và quan sát pipeline thất bại; sau đó khôi phục test thành công.

**Troubleshooting:** Kiểm tra log khởi động controller và output của job bằng `docker logs --tail 100 jenkins` cùng mục **Console Output** của Jenkins job. Kiểm tra health/state của container bằng `docker ps -a --filter name=jenkins`, đồng thời xác minh file được mount bằng `docker exec jenkins ls -la /workspace`. Console của job cho biết stage và command bị lỗi.

---
