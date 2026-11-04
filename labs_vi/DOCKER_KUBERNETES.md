# Docker, Compose và Kubernetes với Minikube

## Tạo và chạy Docker image

```bash
echo "Step 1/4: Define the container image"
cd ~/devops-labs/hello-devops
cat > Dockerfile <<'EOF'
FROM nginx:1.27-alpine
COPY app/ /usr/share/nginx/html/
EXPOSE 80
EOF

echo "Step 2/4: Build the image"
docker build -t hello-devops:1.0 .
echo "Step 3/4: Start the container and request the page"
docker run -d --name hello-devops -p 127.0.0.1:8081:80 hello-devops:1.0
curl --fail http://127.0.0.1:8081/
echo "Step 4/4: Inspect container logs and image metadata"
docker logs hello-devops
docker inspect --format '{{.Config.Image}}' hello-devops
```

**Thử nhé:** Thay đổi HTML, build `hello-devops:1.1`, thay container đang chạy và xác minh phản hồi. Giải thích vì sao thẻ image có phiên bản hữu ích hơn việc chỉ dựa vào `latest`.

**Gỡ lỗi:** Bước cuối cùng được in ra cho biết giai đoạn build, khởi động hay gửi yêu cầu đã gặp lỗi. Với container đang chạy, hãy kiểm tra `docker ps -a`, `docker logs --tail 100 hello-devops`, `docker inspect hello-devops` và `curl -v http://127.0.0.1:8081/`.

## Theo dõi Docker container, image, tiến trình và cổng

Đây là các lệnh kiểm tra chỉ đọc, trừ khi có ghi chú khác. Chúng giúp xác định tiến trình nào đang chạy, sử dụng tài nguyên gì, chiếm cổng nào trên host và được tạo từ image nào.

```bash
echo "Step 1/5: List running and stopped containers"
docker ps
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"
docker ps -a
docker ps --filter name=hello-devops

echo "Step 2/5: Inspect container resource use and processes"
docker stats --no-stream
docker top hello-devops
docker inspect --format 'State={{.State.Status}} Started={{.State.StartedAt}} Image={{.Config.Image}}' hello-devops

echo "Step 3/5: Check container logs and published ports"
docker logs --tail 50 --timestamps hello-devops
docker port hello-devops
docker inspect --format '{{json .NetworkSettings.Ports}}' hello-devops

echo "Step 4/5: List images and inspect image layers"
docker image ls
docker image ls --digests
docker image inspect hello-devops:1.0 --format 'ID={{.Id}} Size={{.Size}} Created={{.Created}}'
docker history hello-devops:1.0
docker system df

echo "Step 5/5: Inspect Docker networks and host listening ports"
docker network ls
docker network inspect bridge
sudo ss -lntup
free -h
df -h
```

`docker stats` nếu không có `--no-stream` sẽ cập nhật liên tục; nhấn `Ctrl-C` để dừng. `docker top` báo cáo các tiến trình trong container từ host. Với container có lệnh `ps`, hãy kiểm tra từ bên trong bằng `docker exec hello-devops ps`; các image tối giản có thể không chứa lệnh đó. `ss` hiển thị các cổng đang lắng nghe trên host; `docker port` hiển thị ánh xạ từ host vào container. Ở đây, `127.0.0.1:8081` nghĩa là cổng ứng dụng chỉ được publish trên giao diện loopback của host.

Theo dõi hoạt động và chẩn đoán các sự cố thường gặp:

```bash
echo "Recent Docker daemon events"
docker events --since 10m

echo "Container state and recent logs"
docker inspect hello-devops --format '{{json .State}}'
docker logs --since 10m --tail 100 hello-devops

echo "Verify the HTTP response and listening socket"
curl -v http://127.0.0.1:8081/
sudo ss -lntp | grep ':8081'
```

`docker events` truyền các sự kiện mới cho đến khi dừng bằng `Ctrl-C`. Nếu cổng trên host đang bị chiếm, `docker run` sẽ báo lỗi bind; xác định tiến trình đang lắng nghe bằng `sudo ss -lntp | grep ':8081'`, rồi dừng container lab đang sử dụng cổng đó hoặc chọn một cổng host khác. Tránh chạy `docker system prune` trong lúc thực hành: lệnh này xóa tài nguyên không dùng đến và có thể xóa dữ liệu hoặc image cần thiết cho các lab khác.

## Định nghĩa và chạy dịch vụ Compose

```bash
echo "Step 1/3: Define the Compose service"
cd ~/devops-labs/hello-devops
cat > compose.yaml <<'EOF'
services:
  web:
    build: .
    image: hello-devops:compose
    ports:
      - "127.0.0.1:8082:80"
    restart: unless-stopped
EOF

echo "Step 2/3: Build and start the Compose service"
docker compose up -d --build
echo "Step 3/3: Request the page and inspect the service"
curl --fail http://127.0.0.1:8082/
docker compose ps
echo "Cleaning up the Compose service"
docker compose down
```

**Gỡ lỗi:** Chạy `docker compose ps`, `docker compose logs --tail 100 web` và `docker compose config` từ thư mục chứa `compose.yaml`. `docker compose logs -f web` theo dõi đầu ra trực tiếp; nhấn `Ctrl-C` để dừng theo dõi.

Xem trạng thái các dịch vụ theo cách riêng của Compose:

```bash
echo "Compose service state, resource use, and recent events"
docker compose ps --all
docker compose top
docker compose logs --since 10m --tail 100
docker compose events
```

`docker compose events` truyền các sự kiện cho đến khi dừng bằng `Ctrl-C`. Nếu `docker compose top` không hiển thị tiến trình nào, hãy kiểm tra dịch vụ có đang chạy bằng `docker compose ps` và xem log của dịch vụ.

## Triển khai và vận hành ứng dụng trên Minikube

Khởi động một cluster cục bộ một node và nạp image vừa build vào đó:

```bash
echo "Step 1/3: Start Minikube"
minikube start --driver=docker --cpus=2 --memory=4096
echo "Step 2/3: Verify the cluster node"
kubectl get nodes
echo "Step 3/3: Load the application image into Minikube"
minikube image load hello-devops:1.0
```

Tạo các tài nguyên Kubernetes:

```bash
echo "Step 1/4: Create and apply a Deployment"
kubectl create deployment hello-devops --image=hello-devops:1.0 \
  --dry-run=client -o yaml > /tmp/hello-deployment.yaml
kubectl apply -f /tmp/hello-deployment.yaml
echo "Step 2/4: Expose the application and wait for rollout"
kubectl expose deployment hello-devops --type=NodePort --port=80
kubectl rollout status deployment/hello-devops
echo "Step 3/4: Inspect resources and get the service URL"
kubectl get deployments,pods,services
echo "Step 4/4: Open the service URL"
minikube service hello-devops --url
```

Dùng URL được in ra với `curl` từ host Ubuntu. Thử mở rộng quy mô và xem lịch sử rollout:

```bash
echo "Step 1/3: Scale the deployment and inspect pod placement"
kubectl scale deployment/hello-devops --replicas=3
kubectl get pods -o wide
kubectl rollout history deployment/hello-devops
echo "Step 2/3: Build and load the updated image"
docker build -t hello-devops:1.1 .
minikube image load hello-devops:1.1
echo "Step 3/3: Update the deployment and wait for rollout"
kubectl set image deployment/hello-devops hello-devops=hello-devops:1.1
kubectl rollout status deployment/hello-devops
```

Nếu rollout thất bại, hãy kiểm tra và khôi phục phiên bản trước:

```bash
echo "Step 1/3: Inspect deployment status and recent events"
kubectl describe deployment hello-devops
kubectl get events --sort-by=.metadata.creationTimestamp
echo "Step 2/3: Roll back to the previous image"
kubectl rollout undo deployment/hello-devops
echo "Step 3/3: Verify the rollback"
kubectl rollout status deployment/hello-devops
```

**Xác minh:** Ba pod đạt trạng thái `Running`; dịch vụ trả về trang; bản cập nhật image hợp lệ được rollout; và có thể chẩn đoán cũng như khôi phục khi dùng tag không hợp lệ.

Để xem đầu ra pod trực tiếp, dùng `kubectl logs -f deployment/hello-devops --all-pods=true`; nhấn `Ctrl-C` để dừng theo dõi. Nếu pod chưa sẵn sàng, hãy kiểm tra `kubectl describe pod <pod-name>`, `kubectl get events --sort-by=.metadata.creationTimestamp` và `minikube logs --problems`.

### Theo dõi workload và dịch vụ Kubernetes

Các lệnh này cung cấp góc nhìn Kubernetes về trạng thái mong muốn và thực tế, mức sử dụng tài nguyên, endpoint dịch vụ và các sự kiện gần đây của cluster:

```bash
echo "Step 1/3: Inspect nodes, deployments, pods, and service endpoints"
kubectl get nodes -o wide
kubectl get deployments,pods,services -o wide
kubectl get endpoints hello-devops

echo "Step 2/3: Watch pod changes and inspect recent events"
kubectl get pods --watch
```

Dừng `kubectl get pods --watch` bằng `Ctrl-C`, sau đó kiểm tra chỉ số tài nguyên:

```bash
echo "Step 3/3: Enable metrics and view node and pod resource use"
minikube addons enable metrics-server
kubectl top nodes
kubectl top pods --all-namespaces
```

Chỉ số có thể cần một chút thời gian mới xuất hiện sau khi bật addon. Để xem log ứng dụng, dùng `kubectl logs -f deployment/hello-devops --all-pods=true`; để xem sự kiện cluster mới, dùng `kubectl get events --watch` và dừng bằng `Ctrl-C`. `kubectl describe pod <pod-name>` bao gồm thông tin về lập lịch, probe và kéo image.

### Theo dõi toàn bộ Kubernetes cluster

Chạy các lệnh này từ host Ubuntu để kiểm tra cluster đang hoạt động, API server, các node và workload hệ thống:

```bash
echo "Step 1/5: Check the selected Minikube cluster and Kubernetes API"
kubectl config current-context
minikube status
kubectl cluster-info
kubectl get --raw='/readyz?verbose'

echo "Step 2/5: Check node readiness, addresses, and resource capacity"
kubectl get nodes -o wide
kubectl describe nodes
kubectl top nodes

echo "Step 3/5: Inspect workloads in every namespace"
kubectl get namespaces
kubectl get pods -A -o wide
kubectl get deployments,statefulsets,daemonsets -A
kubectl get services,endpointslices -A

echo "Step 4/5: Review cluster-wide events and API extensions"
kubectl get events -A --sort-by=.metadata.creationTimestamp
kubectl get apiservices

echo "Step 5/5: Check storage and scheduling constraints"
kubectl get storageclasses,persistentvolumes
kubectl get persistentvolumeclaims -A
kubectl get resourcequotas,limitranges -A
```

Để theo dõi liên tục, chạy riêng từng lệnh này trong terminal và dừng bằng `Ctrl-C`:

```bash
kubectl get nodes --watch
kubectl get pods -A --watch
kubectl get events -A --watch
kubectl top pods -A --containers
```

Dùng tên node từ `kubectl get nodes` để chẩn đoán tập trung:

```bash
kubectl describe node <node-name>
kubectl get pods -A --field-selector=status.phase!=Running
kubectl get events -A --sort-by=.metadata.creationTimestamp
minikube logs --problems
```

`kubectl top` cần Metrics API; trên Minikube, bật API này bằng `minikube addons enable metrics-server` và chờ một chút để chỉ số xuất hiện. Yêu cầu `readyz` kiểm tra API server đã sẵn sàng hay chưa, còn `minikube status` kiểm tra các thành phần Minikube cục bộ. Khi pod gặp lỗi, dùng `kubectl describe pod -n <namespace> <pod-name>` và `kubectl logs -n <namespace> <pod-name> --previous` nếu container đã khởi động lại. Pod ở trạng thái Pending thường cho thấy yêu cầu storage hoặc CPU/memory chưa được đáp ứng; hãy kiểm tra PVC, dung lượng node, sự kiện và đầu ra của `kubectl describe pod`.

## Triển khai Jenkins trên Minikube

Lab này tạo Jenkins trong namespace riêng, với persistent claim và dịch vụ ClusterIP nội bộ. Truy cập giao diện bằng `kubectl port-forward` để không expose dịch vụ ra mạng của host. StorageClass mặc định của Minikube tự động cấp phát claim.

Tạo namespace và storage claim:

```bash
echo "Step 1/4: Create the CI namespace and persistent Jenkins storage"
kubectl create namespace ci --dry-run=client -o yaml | kubectl apply -f -
cat > /tmp/jenkins-pvc.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-home
  namespace: ci
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
EOF
kubectl apply -f /tmp/jenkins-pvc.yaml
kubectl get pvc -n ci
```

Triển khai Jenkins và chờ dịch vụ sẵn sàng:

```bash
echo "Step 2/4: Create the Jenkins Deployment and internal Service"
cat > /tmp/jenkins.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: ci
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      securityContext:
        fsGroup: 1000
      containers:
        - name: jenkins
          image: jenkins/jenkins:lts-jdk17
          ports:
            - name: http
              containerPort: 8080
            - name: agent
              containerPort: 50000
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
            limits:
              memory: 2Gi
          readinessProbe:
            httpGet:
              path: /login
              port: http
            initialDelaySeconds: 60
            periodSeconds: 10
          volumeMounts:
            - name: jenkins-home
              mountPath: /var/jenkins_home
      volumes:
        - name: jenkins-home
          persistentVolumeClaim:
            claimName: jenkins-home
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: ci
spec:
  type: ClusterIP
  selector:
    app: jenkins
  ports:
    - name: http
      port: 8080
      targetPort: http
    - name: agent
      port: 50000
      targetPort: agent
EOF
kubectl apply -f /tmp/jenkins.yaml
echo "Step 3/4: Wait for Jenkins rollout and inspect resources"
kubectl rollout status deployment/jenkins -n ci --timeout=5m
kubectl get deployment,pods,services,pvc -n ci -o wide
```

Forward cổng giao diện và lấy mật khẩu thiết lập dùng một lần:

```bash
echo "Step 4/4: Readiness, logs, setup password, and local UI access"
kubectl logs -n ci deployment/jenkins --tail=80
kubectl exec -n ci deployment/jenkins -- \
  cat /var/jenkins_home/secrets/initialAdminPassword
kubectl port-forward -n ci service/jenkins 8080:8080
```

Giữ lệnh port-forward chạy trong khi mở `http://127.0.0.1:8080` bằng trình duyệt. Dừng port-forward bằng `Ctrl-C`. Hoàn tất thiết lập Jenkins và cài plugin Pipeline. Mật khẩu quản trị là thông tin nhạy cảm; không dán mật khẩu vào log, source control hoặc terminal dùng chung.

Kiểm tra trạng thái Jenkins và Kubernetes:

```bash
echo "Jenkins health and diagnostic commands"
kubectl get pods -n ci -o wide
kubectl describe pod -n ci -l app=jenkins
kubectl logs -n ci deployment/jenkins --since=10m --tail=200
kubectl get events -n ci --sort-by=.metadata.creationTimestamp
kubectl get pvc -n ci
```

**Gỡ lỗi:** Nếu pod ở trạng thái `Pending`, hãy kiểm tra PVC đã bind chưa, dung lượng node và sự kiện trong namespace. Nếu pod liên tục khởi động lại, xem `kubectl logs -n ci deployment/jenkins --previous --tail=200`. Nếu Jenkins khởi động chậm, kiểm tra log hiện tại và chờ readiness probe; Jenkins có thể cần thêm bộ nhớ, hãy kiểm tra `kubectl describe pod -n ci -l app=jenkins` để tìm `OOMKilled` hoặc sự kiện lập lịch. Không scale claim home Jenkins kiểu single-writer này thành nhiều replica.

## Triển khai ứng dụng Node.js đơn giản

Tạo một HTTP server nhỏ có health endpoint, build image và nạp trực tiếp vào kho image của Minikube:

```bash
echo "Step 1/4: Create the Node.js application"
mkdir -p ~/devops-labs/nodejs-demo
cd ~/devops-labs/nodejs-demo
cat > server.js <<'EOF'
const http = require("node:http");

const server = http.createServer((request, response) => {
  if (request.url === "/health") {
    response.writeHead(200, { "content-type": "text/plain" });
    response.end("ok\n");
    return;
  }

  response.writeHead(200, { "content-type": "text/plain" });
  response.end("Hello from Node.js on Minikube\n");
});

server.listen(3000, "0.0.0.0", () => {
  console.log("Listening on port 3000");
});
EOF
cat > Dockerfile <<'EOF'
FROM node:22-alpine
WORKDIR /app
COPY server.js .
USER node
EXPOSE 3000
CMD ["node", "server.js"]
EOF

echo "Step 2/4: Build and load the Node.js image into Minikube"
docker build -t nodejs-demo:1.0 .
minikube image load nodejs-demo:1.0
```

Tạo Deployment có health probe và dịch vụ ClusterIP:

```bash
echo "Step 3/4: Deploy the application and expose it inside the cluster"
cat > /tmp/nodejs-demo.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-demo
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nodejs-demo
  template:
    metadata:
      labels:
        app: nodejs-demo
    spec:
      containers:
        - name: nodejs
          image: nodejs-demo:1.0
          imagePullPolicy: Never
          ports:
            - name: http
              containerPort: 3000
          readinessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 2
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: nodejs-demo
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: nodejs-demo
  ports:
    - name: http
      port: 3000
      targetPort: http
EOF
kubectl apply -f /tmp/nodejs-demo.yaml
kubectl rollout status deployment/nodejs-demo --timeout=2m
kubectl get deployment,pods,service -l app=nodejs-demo -o wide
```

Dùng port-forward để kiểm thử ứng dụng từ host Ubuntu:

```bash
echo "Step 4/4: Forward the service port; run curl from another terminal"
kubectl port-forward service/nodejs-demo 3000:3000
```

Trong khi port-forward đang chạy, hãy dùng terminal khác:

```bash
curl --fail http://127.0.0.1:3000/
curl --fail http://127.0.0.1:3000/health
kubectl logs deployment/nodejs-demo --all-pods=true --tail=100
kubectl get events --sort-by=.metadata.creationTimestamp
```

Dừng port-forward bằng `Ctrl-C`. Để cập nhật trực tiếp, thay đổi nội dung phản hồi, build lại thành `nodejs-demo:1.1`, nạp bằng `minikube image load nodejs-demo:1.1`, rồi chạy:

```bash
kubectl set image deployment/nodejs-demo nodejs=nodejs-demo:1.1
kubectl rollout status deployment/nodejs-demo
kubectl rollout history deployment/nodejs-demo
```

**Gỡ lỗi:** `kubectl describe pod -l app=nodejs-demo` hiển thị lỗi về image, probe và lập lịch. `kubectl logs deployment/nodejs-demo --all-pods=true` hiển thị đầu ra ứng dụng; sau khi pod khởi động lại, dùng `--previous` với một pod cụ thể. Nếu không sử dụng image mới build cục bộ, hãy nạp image đó vào Minikube và dùng tag phiên bản mới thay vì dựa vào tag đã được cache.

## Triển khai Open WebUI trên Minikube

Lab này chỉ triển khai Open WebUI. **Model provider là tùy chọn**: bạn có thể kết nối đến model API được quản lý riêng, hoặc tiếp tục với phần Ollama tùy chọn trong cluster bên dưới. Giao diện chỉ được expose qua port-forward cục bộ; không expose môi trường thực hành này ra công khai.

Xem [hướng dẫn bắt đầu nhanh Open WebUI](https://docs.openwebui.com/getting-started/quick-start/) và [tài liệu Ollama](https://github.com/ollama/ollama/blob/main/docs/README.md) để biết thông tin sản phẩm.

Tạo namespace và secret sinh tự động cho signing key của Open WebUI. Giữ nguyên key qua các lần pod khởi động lại; Kubernetes Secret là cấu hình được mã hóa base64, không phải kho bí mật đã mã hóa.

```bash
echo "Step 1/4: Create the AI namespace and WebUI secret"
kubectl create namespace ai --dry-run=client -o yaml | kubectl apply -f -
if kubectl get secret open-webui-secrets -n ai >/dev/null 2>&1; then
  echo "Reusing the existing WebUI signing key"
else
  kubectl create secret generic open-webui-secrets -n ai \
    --from-literal=WEBUI_SECRET_KEY="$(openssl rand -hex 32)"
fi
```

Tạo storage bền vững và Service nội bộ cho Open WebUI:

```bash
echo "Step 2/4: Define persistent storage and the internal WebUI Service"
cat > /tmp/open-webui.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: open-webui-data
  namespace: ai
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
---
apiVersion: v1
kind: Service
metadata:
  name: open-webui
  namespace: ai
spec:
  type: ClusterIP
  selector:
    app: open-webui
  ports:
    - name: http
      port: 8080
      targetPort: http
EOF
kubectl apply -f /tmp/open-webui.yaml
kubectl get pvc,services -n ai
```

Triển khai Open WebUI với dữ liệu ứng dụng được lưu bền vững:

```bash
echo "Step 3/4: Deploy Open WebUI"
cat > /tmp/open-webui-deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: open-webui
  namespace: ai
spec:
  replicas: 1
  selector:
    matchLabels:
      app: open-webui
  template:
    metadata:
      labels:
        app: open-webui
    spec:
      containers:
        - name: open-webui
          image: ghcr.io/open-webui/open-webui:main-slim
          env:
            - name: WEBUI_SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: open-webui-secrets
                  key: WEBUI_SECRET_KEY
          ports:
            - name: http
              containerPort: 8080
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              memory: 2Gi
          startupProbe:
            httpGet:
              path: /health
              port: http
            periodSeconds: 10
            failureThreshold: 30
          readinessProbe:
            httpGet:
              path: /health
              port: http
            periodSeconds: 10
          volumeMounts:
            - name: open-webui-data
              mountPath: /app/backend/data
      volumes:
        - name: open-webui-data
          persistentVolumeClaim:
            claimName: open-webui-data
EOF
kubectl apply -f /tmp/open-webui-deployment.yaml
echo "Waiting for Open WebUI to become ready"
kubectl rollout status deployment/open-webui -n ai --timeout=5m
kubectl get deployments,pods,pvc,services -n ai -o wide
```

Forward Open WebUI tới host Ubuntu:

```bash
echo "Step 4/4: Check the logs and open WebUI locally"
kubectl logs -n ai deployment/open-webui --tail=80
kubectl port-forward -n ai service/open-webui 3000:8080
```

Trong khi port-forward đang chạy, mở `http://127.0.0.1:3000` bằng trình duyệt. Tạo tài khoản đầu tiên; tài khoản này sẽ trở thành quản trị viên. Bạn có thể dùng giao diện mà chưa cần kết nối model để khám phá các thiết lập tài khoản và quản trị. Để trò chuyện, cấu hình model provider trong phần cài đặt kết nối của quản trị viên. Dừng port-forward bằng `Ctrl-C`.

### Tùy chọn: Chạy Ollama trong cluster

Bỏ qua phần này nếu bạn đã có model API endpoint, không muốn suy luận model cục bộ hoặc Minikube bị giới hạn bộ nhớ hay dung lượng đĩa. Ollama tải model riêng và sử dụng thêm CPU, bộ nhớ cùng storage bền vững.

Tạo storage Ollama, Service nội bộ và Deployment một replica:

```bash
echo "Optional step 1/3: Create Ollama storage and API service"
cat > /tmp/ollama.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ollama-data
  namespace: ai
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
---
apiVersion: v1
kind: Service
metadata:
  name: ollama
  namespace: ai
spec:
  type: ClusterIP
  selector:
    app: ollama
  ports:
    - name: api
      port: 11434
      targetPort: api
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ollama
  namespace: ai
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ollama
  template:
    metadata:
      labels:
        app: ollama
    spec:
      containers:
        - name: ollama
          image: ollama/ollama:latest
          ports:
            - name: api
              containerPort: 11434
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
            limits:
              memory: 3Gi
          startupProbe:
            httpGet:
              path: /api/tags
              port: api
            periodSeconds: 10
            failureThreshold: 30
          readinessProbe:
            httpGet:
              path: /api/tags
              port: api
            periodSeconds: 10
          volumeMounts:
            - name: ollama-data
              mountPath: /root/.ollama
      volumes:
        - name: ollama-data
          persistentVolumeClaim:
            claimName: ollama-data
EOF
kubectl apply -f /tmp/ollama.yaml
kubectl rollout status deployment/ollama -n ai --timeout=5m
```

Tải xuống và xác minh một model nhỏ:

```bash
echo "Optional step 2/3: Download a small model and verify it is available"
kubectl exec -n ai deployment/ollama -- ollama pull qwen3:0.6b
kubectl exec -n ai deployment/ollama -- ollama list
```

Trong phần cài đặt kết nối của quản trị viên Open WebUI, thêm kết nối Ollama bằng `http://ollama:11434`, lưu lại và chọn `qwen3:0.6b` trong cuộc trò chuyện mới. URL này là tên DNS của Kubernetes Service và hoạt động từ bên trong cluster.

Khi khắc phục sự cố, hãy kiểm tra tài nguyên cluster và log ứng dụng:

```bash
echo "Optional step 3/3: Inspect Open WebUI and Ollama diagnostics"
kubectl get pods,services,endpoints,pvc -n ai -o wide
kubectl top pods -n ai --containers
kubectl logs -n ai deployment/open-webui --since=10m --tail=200
kubectl get events -n ai --sort-by=.metadata.creationTimestamp
```

Nếu đã cài Ollama, hãy kiểm tra cả log và danh sách model:

```bash
kubectl logs -n ai deployment/ollama --since=10m --tail=200
kubectl exec -n ai deployment/ollama -- ollama list
```

**Gỡ lỗi:** Nếu PVC ở trạng thái `Pending`, hãy kiểm tra `kubectl get storageclass` và sự kiện trong namespace. Nếu giao diện không kết nối được với provider bên ngoài, hãy kiểm tra URL provider và khả năng kết nối từ cluster. Với sự cố Ollama, kiểm tra pod, endpoint dịch vụ và log; nếu tải model hoặc suy luận thất bại, kiểm tra bộ nhớ pod bằng `kubectl top pods -n ai --containers` và tìm sự kiện `OOMKilled`, sau đó dùng model nhỏ hơn hoặc tăng bộ nhớ Minikube. Database và dữ liệu trò chuyện của Open WebUI vẫn tồn tại sau khi pod khởi động lại vì chúng dùng PVC. Dữ liệu Ollama chỉ được lưu bền vững khi cài backend tùy chọn.

**Dọn dẹp:** Xóa namespace để gỡ Open WebUI, secret và dữ liệu trò chuyện đã lưu. Nếu đã cài Ollama tùy chọn, thao tác này cũng xóa các model đã tải xuống.

```bash
kubectl delete namespace ai
```

## Triển khai RAGFlow trên Minikube (Community Helm Chart)

> **Chỉ dùng để học / thử nghiệm:** [Hướng dẫn triển khai upstream](https://github.com/infiniflow/ragflow/blob/main/docker/README.md) của RAGFlow mô tả Docker Compose, không phải chart Kubernetes này. Lab sử dụng [RAGFlow Helm chart](https://github.com/whg517/ragflow-helm-chart) do cộng đồng duy trì, được cố định tại [commit `78ecb423`](https://github.com/whg517/ragflow-helm-chart/commit/78ecb423416a852857a78b4fcb16e57b23d88bb1). Chart này nhắm đến RAGFlow `v0.27.0`, không phải dòng phát hành mới hơn dựa trên Go. Hãy xem xét chart và các dependency trước khi cài; đây không phải bản triển khai được upstream hỗ trợ hay dành cho production.

RAGFlow cần một số dịch vụ stateful. Chart đánh giá một node này bật PostgreSQL, Valkey, lưu trữ object RustFS và OpenSearch cùng với RAGFlow. Hãy dự kiến tải image lớn và sử dụng nhiều dung lượng đĩa/RAM. Dùng Minikube profile dùng một lần với ít nhất 16 GB RAM trên host, 6 CPU core và 50 GB dung lượng đĩa trống; cấp cho Minikube 12 GB RAM và 6 CPU. Trước tiên, hãy dừng Jenkins, Ollama và các dịch vụ lab khác ngốn nhiều bộ nhớ. Không tải tài liệu nhạy cảm lên và không dùng thông tin xác thực production thật.

Cài Helm nếu máy chưa có và chuẩn bị node Minikube cho OpenSearch:

```bash
echo "Step 1/5: Install Helm and check the cluster"
if ! command -v helm >/dev/null 2>&1; then
  sudo snap install helm --classic
fi
helm version
kubectl config current-context

echo "Step 2/5: Configure Minikube resources and OpenSearch kernel setting"
sudo sysctl -w vm.max_map_count=262144
minikube stop
minikube start --driver=docker --cpus=6 --memory=12288 --disk-size=60g
minikube status
```

Image RAGFlow mặc định của community chart chạy dưới quyền root; lab này dành cho cluster cục bộ đáng tin cậy và không đáp ứng chính sách restricted-Pod-Security nghiêm ngặt. Không expose dịch vụ ra mạng không đáng tin cậy.

`vm.max_map_count` là thiết lập kernel trên host mà OpenSearch yêu cầu; thay đổi bằng `sysctl` chỉ có hiệu lực tạm thời và sẽ đặt lại sau khi host khởi động lại. Nếu Helm đã được cài, hãy bỏ qua lệnh snap. Nếu host không thể cung cấp đủ tài nguyên, hãy bỏ qua lab này thay vì làm thiếu tài nguyên cho các dịch vụ khác.

Clone và cố định phiên bản community chart, sau đó kiểm tra và giải quyết các Helm dependency:

```bash
echo "Step 3/5: Clone the chart at the reviewed commit and build dependencies"
git clone https://github.com/whg517/ragflow-helm-chart.git \
  ~/devops-labs/ragflow-helm-chart
cd ~/devops-labs/ragflow-helm-chart
git checkout 78ecb423416a852857a78b4fcb16e57b23d88bb1
git status --short --branch
cd ragflow
helm dependency build
helm lint .
```

Tạo thông tin xác thực dưới dạng Kubernetes Secret. Các lệnh sẽ nhắc nhập giá trị để mật khẩu dạng plaintext không bị nhúng vào file values của chart. Lưu mật khẩu admin ban đầu trong password manager; không in Secret bằng `kubectl get secret -o yaml`.

```bash
echo "Step 4/5: Create the RAGFlow namespace and credentials Secret"
kubectl create namespace ragflow --dry-run=client -o yaml | kubectl apply -f -
read -rsp "RAGFlow initial admin password: " RAGFLOW_ADMIN_PASSWORD
echo
kubectl create secret generic ragflow-credentials -n ragflow \
  --from-literal=MYSQL_PASSWORD="$(openssl rand -hex 24)" \
  --from-literal=REDIS_PASSWORD="$(openssl rand -hex 24)" \
  --from-literal=MINIO_USER=rag_flow \
  --from-literal=MINIO_PASSWORD="$(openssl rand -hex 24)" \
  --from-literal=OPENSEARCH_PASSWORD="$(openssl rand -hex 24)" \
  --from-literal=DEFAULT_SUPERUSER_EMAIL=lab-admin@example.invalid \
  --from-literal=DEFAULT_SUPERUSER_PASSWORD="$RAGFLOW_ADMIN_PASSWORD" \
  --dry-run=client -o yaml | kubectl apply -f -
unset RAGFLOW_ADMIN_PASSWORD
```

Cấu hình bản triển khai đánh giá có quy mô nhỏ, một node. Các dịch vụ stateful tích hợp trong chart không có tính sẵn sàng cao. Chart đi kèm schema values riêng; hãy render và kiểm tra manifest trước khi áp dụng:

```bash
echo "Creating evaluation values with one replica and bounded resources"
cat > /tmp/ragflow-minikube-values.yaml <<'EOF'
image:
  repository: infiniflow/ragflow
  tag: v0.27.0
  pullPolicy: IfNotPresent

existingSecret: ragflow-credentials

metadataDb:
  type: postgres
  user: ragflow
  database: rag_flow
  postgres:
    builtin:
      enabled: true
      persistence:
        size: 8Gi
      resources:
        requests:
          cpu: 250m
          memory: 512Mi
        limits:
          cpu: "1"
          memory: 1Gi

valkey:
  enabled: true

storage:
  type: minio
  minio:
    user: rag_flow
    bucket: rag_flow

rustfs:
  enabled: true
  persistence:
    enabled: true
    size: 8Gi
  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: "1"
      memory: 1Gi

docEngine:
  type: opensearch

opensearch:
  enabled: true
  singleNode: true
  resources:
    requests:
      cpu: "1"
      memory: 2Gi
    limits:
      memory: 3Gi
  opensearchJavaOpts: "-Xms1g -Xmx1g"
  persistence:
    enabled: true
    size: 8Gi

api:
  replicaCount: 1
  initSuperuser: true
  registerEnabled: false
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: "2"
      memory: 3Gi

executor:
  replicaCount: 1
  workers: 1
  resources:
    requests:
      cpu: "1"
      memory: 2Gi
    limits:
      cpu: "2"
      memory: 4Gi

datasync:
  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: "1"
      memory: 2Gi

migration:
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: "2"
      memory: 2Gi
EOF

echo "Render manifests for review; do not proceed if Helm reports an error"
helm template ragflow . --namespace ragflow \
  --values /tmp/ragflow-minikube-values.yaml > /tmp/ragflow-rendered.yaml
kubectl apply --dry-run=client -f /tmp/ragflow-rendered.yaml
```

Cài chart và chờ hoàn tất. Lần chạy đầu tiên tải các image có dung lượng nhiều GB nên có thể mất nhiều thời gian:

```bash
echo "Step 5/5: Install RAGFlow and inspect rollout"
helm upgrade --install ragflow . --namespace ragflow \
  --values /tmp/ragflow-minikube-values.yaml \
  --wait --timeout 30m
kubectl get pods,services,pvc -n ragflow -o wide
kubectl get jobs -n ragflow
kubectl rollout status deployment/ragflow-api -n ragflow --timeout=10m
```

Nếu tên tài nguyên trong chart khác với dự kiến, dùng `kubectl get deployments,statefulsets,pods -n ragflow` để xác định; tên release Helm sẽ được thêm vào trước tên các tài nguyên được tạo.

Forward cổng dịch vụ web RAGFlow và xác minh API:

```bash
kubectl get services -n ragflow
kubectl port-forward -n ragflow service/ragflow-api 8088:80
```

Trong khi port-forward đang chạy, mở `http://127.0.0.1:8088` bằng trình duyệt và đăng nhập bằng `lab-admin@example.invalid` cùng mật khẩu đã nhập ở trên. Dừng port-forward bằng `Ctrl-C`. Cấu hình LLM và embedding provider trong phần cài đặt model của RAGFlow trước khi tạo knowledge base; chart Kubernetes không cài language model. Hãy dùng provider/API key dành cho lab và tránh tải dữ liệu mật lên.

Kiểm tra workload và dependency của chart:

```bash
echo "RAGFlow cluster diagnostics"
kubectl get pods -n ragflow -o wide
kubectl get deployments,statefulsets,services,pvc -n ragflow
minikube addons enable metrics-server
# Metrics may take a short time to become available after enabling the addon.
kubectl top pods -n ragflow --containers
kubectl get events -n ragflow --sort-by=.metadata.creationTimestamp
helm status ragflow -n ragflow
helm history ragflow -n ragflow
```

Để xem log, thay `<pod-name>` bằng tên pod lấy từ `kubectl get pods -n ragflow`:

```bash
kubectl describe pod -n ragflow <pod-name>
kubectl logs -n ragflow <pod-name> --all-containers=true --tail=200
kubectl logs -n ragflow <pod-name> --all-containers=true --previous --tail=200
```

**Gỡ lỗi:** OpenSearch thường không khởi động được khi `vm.max_map_count` quá thấp; xác minh bằng `sysctl vm.max_map_count`. Pod ở trạng thái `Pending` có thể cho thấy Minikube không đủ bộ nhớ, CPU hoặc dung lượng persistent disk. Dùng sự kiện, mô tả pod và trạng thái PVC để xác định dependency nào bị chặn. Health check readiness của RAGFlow cũng xác minh database, cache, document engine và object storage; hãy chờ mọi dependency sẵn sàng trước khi chẩn đoán API pod. Tránh chạy `helm uninstall` cho đến khi quyết định có giữ lại dữ liệu persistent trong namespace hay không.

**Dọn dẹp:** Thao tác này gỡ chart và các dependency demo trong namespace. Cách thu hồi persistent volume phụ thuộc vào StorageClass của Minikube; hãy kiểm tra PVC và PV trước khi xóa nếu cần giữ dữ liệu lab.

```bash
helm uninstall ragflow -n ragflow
kubectl delete namespace ragflow
```

## Dọn dẹp

Khi hoàn tất, hãy gỡ các tài nguyên Kubernetes của lab. Xóa Jenkins PVC hoặc namespace `ci` sẽ xóa cấu hình Jenkins đã lưu; hãy export hoặc giữ lại dữ liệu trước khi dọn dẹp nếu muốn bảo toàn instance.

```bash
echo "Step 1/2: Remove the Node.js application"
kubectl delete -f /tmp/nodejs-demo.yaml --ignore-not-found

echo "Step 2/2: Remove Jenkins and its persistent data"
kubectl delete -f /tmp/jenkins.yaml --ignore-not-found
kubectl delete -f /tmp/jenkins-pvc.yaml --ignore-not-found
kubectl delete namespace ci --ignore-not-found
```

---
