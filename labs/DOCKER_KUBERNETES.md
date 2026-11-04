# Docker, Compose, and Kubernetes with Minikube

## Build and Run a Docker Image

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

**Try it:** Change the HTML, build `hello-devops:1.1`, replace the running container, and verify the response. Explain why a versioned image tag is more useful than relying on `latest`.

**Debug:** The last printed step marker identifies the build, startup, or request phase that failed. For a running container, check `docker ps -a`, `docker logs --tail 100 hello-devops`, `docker inspect hello-devops`, and `curl -v http://127.0.0.1:8081/`.

## Monitor Docker Containers, Images, Processes, and Ports

These are read-only inspection commands unless noted. They help identify what is running, what resources it uses, which host ports it occupies, and which image it came from.

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

`docker stats` without `--no-stream` updates continuously; press `Ctrl-C` to stop it. `docker top` reports processes in the container from the host. For a container that includes `ps`, inspect from inside with `docker exec hello-devops ps`; minimal images may not include that command. `ss` shows host listeners; `docker port` shows the host-to-container mapping. Here, `127.0.0.1:8081` means the application port is only published on the host's loopback interface.

Follow activity and diagnose common issues:

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

`docker events` streams new events until stopped with `Ctrl-C`. If the host port is occupied, `docker run` reports a bind error; identify the listener with `sudo ss -lntp | grep ':8081'` and either stop the lab container that owns it or choose another host port. Avoid `docker system prune` during exercises: it deletes unused resources and can remove data or images needed by other labs.

## Define and Run a Compose Service

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

**Debug:** Run `docker compose ps`, `docker compose logs --tail 100 web`, and `docker compose config` from the directory containing `compose.yaml`. `docker compose logs -f web` follows live output; press `Ctrl-C` to stop following.

For a Compose-specific view across the services:

```bash
echo "Compose service state, resource use, and recent events"
docker compose ps --all
docker compose top
docker compose logs --since 10m --tail 100
docker compose events
```

`docker compose events` streams events until stopped with `Ctrl-C`. If `docker compose top` shows no process, check that the service is running with `docker compose ps` and inspect its logs.

## Deploy and Operate the Application on Minikube

Start a single-node local cluster and load the locally built image into it:

```bash
echo "Step 1/3: Start Minikube"
minikube start --driver=docker --cpus=2 --memory=4096
echo "Step 2/3: Verify the cluster node"
kubectl get nodes
echo "Step 3/3: Load the application image into Minikube"
minikube image load hello-devops:1.0
```

Create Kubernetes resources:

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

Use the printed URL with `curl` from the Ubuntu host. Explore scaling and rollout history:

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

If the rollout fails, inspect it and roll back:

```bash
echo "Step 1/3: Inspect deployment status and recent events"
kubectl describe deployment hello-devops
kubectl get events --sort-by=.metadata.creationTimestamp
echo "Step 2/3: Roll back to the previous image"
kubectl rollout undo deployment/hello-devops
echo "Step 3/3: Verify the rollback"
kubectl rollout status deployment/hello-devops
```

**Verify:** Three pods reach `Running`; the service returns the page; a valid image update rolls out; and a bad tag can be diagnosed and rolled back.

For live pod output use `kubectl logs -f deployment/hello-devops --all-pods=true`; press `Ctrl-C` to stop following. If a pod is not ready, inspect `kubectl describe pod <pod-name>`, `kubectl get events --sort-by=.metadata.creationTimestamp`, and `minikube logs --problems`.

### Monitor Kubernetes Workloads and Services

These commands provide the Kubernetes view of desired and actual state, resource use, service endpoints, and recent cluster events:

```bash
echo "Step 1/3: Inspect nodes, deployments, pods, and service endpoints"
kubectl get nodes -o wide
kubectl get deployments,pods,services -o wide
kubectl get endpoints hello-devops

echo "Step 2/3: Watch pod changes and inspect recent events"
kubectl get pods --watch
```

Stop `kubectl get pods --watch` with `Ctrl-C`, then inspect resource metrics:

```bash
echo "Step 3/3: Enable metrics and view node and pod resource use"
minikube addons enable metrics-server
kubectl top nodes
kubectl top pods --all-namespaces
```

Metrics may take a short time to appear after enabling the addon. For application logs use `kubectl logs -f deployment/hello-devops --all-pods=true`; for new cluster events use `kubectl get events --watch` and stop it with `Ctrl-C`. `kubectl describe pod <pod-name>` includes scheduling, probe, and image-pull details.

### Monitor the Whole Kubernetes Cluster

Run these commands from the Ubuntu host to check the active cluster, API server, nodes, and system workloads:

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

For ongoing observation, run these individually in a terminal and stop them with `Ctrl-C`:

```bash
kubectl get nodes --watch
kubectl get pods -A --watch
kubectl get events -A --watch
kubectl top pods -A --containers
```

Use the node name from `kubectl get nodes` for focused diagnosis:

```bash
kubectl describe node <node-name>
kubectl get pods -A --field-selector=status.phase!=Running
kubectl get events -A --sort-by=.metadata.creationTimestamp
minikube logs --problems
```

`kubectl top` requires the Metrics API; on Minikube enable it with `minikube addons enable metrics-server` and allow a short time for metrics to appear. The `readyz` request checks API-server readiness, while `minikube status` checks local Minikube components. For pod-level failures, use `kubectl describe pod -n <namespace> <pod-name>` and `kubectl logs -n <namespace> <pod-name> --previous` if the container restarted. A Pending pod often points to unsatisfied storage or CPU/memory requests; check the PVC, node capacity, events, and `kubectl describe pod` output.

## Deploy Jenkins on Minikube

This lab creates Jenkins in its own namespace with a persistent claim and an internal ClusterIP service. Access the UI using `kubectl port-forward`, so it is not exposed on the host's network. Minikube's default StorageClass dynamically provisions the claim.

Create the namespace and storage claim:

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

Deploy Jenkins and wait for it to become ready:

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

Forward the UI port and retrieve the one-time setup password:

```bash
echo "Step 4/4: Readiness, logs, setup password, and local UI access"
kubectl logs -n ci deployment/jenkins --tail=80
kubectl exec -n ci deployment/jenkins -- \
  cat /var/jenkins_home/secrets/initialAdminPassword
kubectl port-forward -n ci service/jenkins 8080:8080
```

Keep the port-forward command running while opening `http://127.0.0.1:8080` in a browser. Stop the port-forward with `Ctrl-C`. Complete Jenkins setup and install the Pipeline plugin. The admin password is sensitive; do not paste it into logs, source control, or shared terminals.

Inspect Jenkins and Kubernetes state:

```bash
echo "Jenkins health and diagnostic commands"
kubectl get pods -n ci -o wide
kubectl describe pod -n ci -l app=jenkins
kubectl logs -n ci deployment/jenkins --since=10m --tail=200
kubectl get events -n ci --sort-by=.metadata.creationTimestamp
kubectl get pvc -n ci
```

**Debug:** If the pod is `Pending`, check PVC binding, node capacity, and namespace events. If it is restarting, inspect `kubectl logs -n ci deployment/jenkins --previous --tail=200`. If Jenkins is slow to start, inspect current logs and allow the readiness probe time; Jenkins may need more memory, so check `kubectl describe pod -n ci -l app=jenkins` for `OOMKilled` or scheduling events. Do not scale this single-writer Jenkins home claim to multiple replicas.

## Deploy a Simple Node.js Application

Create a small HTTP server with a health endpoint, build an image, and load it directly into Minikube's image store:

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

Create a Deployment with health probes and a ClusterIP service:

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

Use port-forwarding to test the application from the Ubuntu host:

```bash
echo "Step 4/4: Forward the service port; run curl from another terminal"
kubectl port-forward service/nodejs-demo 3000:3000
```

While port-forward is running, use another terminal:

```bash
curl --fail http://127.0.0.1:3000/
curl --fail http://127.0.0.1:3000/health
kubectl logs deployment/nodejs-demo --all-pods=true --tail=100
kubectl get events --sort-by=.metadata.creationTimestamp
```

Stop port-forward with `Ctrl-C`. For a live update, change the response text, rebuild as `nodejs-demo:1.1`, load it with `minikube image load nodejs-demo:1.1`, then run:

```bash
kubectl set image deployment/nodejs-demo nodejs=nodejs-demo:1.1
kubectl rollout status deployment/nodejs-demo
kubectl rollout history deployment/nodejs-demo
```

**Debug:** `kubectl describe pod -l app=nodejs-demo` shows image, probe, and scheduling errors. `kubectl logs deployment/nodejs-demo --all-pods=true` shows application output; use `--previous` with a specific pod after a restart. If a new locally built image is not used, load it into Minikube and use a new versioned tag rather than relying on a cached tag.

## Deploy Open WebUI on Minikube

This lab deploys Open WebUI by itself. A model provider is **optional**: you can connect to a separately managed model API, or continue with the optional in-cluster Ollama section below. The UI is exposed only through a local port-forward; do not expose this training instance publicly.

See the [Open WebUI quick start](https://docs.openwebui.com/getting-started/quick-start/) and [Ollama documentation](https://github.com/ollama/ollama/blob/main/docs/README.md) for product details.

Create a namespace and a generated secret for the Open WebUI signing key. Keep the key stable across pod restarts; Kubernetes Secrets are base64-encoded configuration, not an encrypted vault.

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

Create persistent storage and an internal Service for Open WebUI:

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

Deploy Open WebUI with persistent application data:

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

Forward Open WebUI to the Ubuntu host:

```bash
echo "Step 4/4: Check the logs and open WebUI locally"
kubectl logs -n ai deployment/open-webui --tail=80
kubectl port-forward -n ai service/open-webui 3000:8080
```

While port-forward is running, open `http://127.0.0.1:3000` in a browser. Create the first account; it becomes the administrator. You can use the UI without connecting a model to explore account and administration settings. To chat, configure a model provider under the administrator's connections settings. Stop port-forward with `Ctrl-C`.

### Optional: Run Ollama in the Cluster

Skip this section if you already have a model API endpoint, do not want local model inference, or Minikube has limited memory or disk. The Ollama model is downloaded separately and uses additional CPU, memory, and persistent storage.

Create Ollama storage, an internal Service, and a single-replica Deployment:

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

Download and verify a small model:

```bash
echo "Optional step 2/3: Download a small model and verify it is available"
kubectl exec -n ai deployment/ollama -- ollama pull qwen3:0.6b
kubectl exec -n ai deployment/ollama -- ollama list
```

In Open WebUI's administrator connection settings, add an Ollama connection using `http://ollama:11434`, save it, and select `qwen3:0.6b` in a new chat. This URL is the Kubernetes Service DNS name and works from inside the cluster.

Inspect the cluster resources and application logs when troubleshooting:

```bash
echo "Optional step 3/3: Inspect Open WebUI and Ollama diagnostics"
kubectl get pods,services,endpoints,pvc -n ai -o wide
kubectl top pods -n ai --containers
kubectl logs -n ai deployment/open-webui --since=10m --tail=200
kubectl get events -n ai --sort-by=.metadata.creationTimestamp
```

When Ollama is installed, also check its logs and model list:

```bash
kubectl logs -n ai deployment/ollama --since=10m --tail=200
kubectl exec -n ai deployment/ollama -- ollama list
```

**Debug:** If a PVC is `Pending`, check `kubectl get storageclass` and namespace events. If the UI cannot reach an external provider, check the provider URL and connectivity from the cluster. For Ollama issues, inspect its pod, service endpoints, and logs; if model download or inference fails, check pod memory with `kubectl top pods -n ai --containers` and events for `OOMKilled`, then use a smaller model or increase Minikube memory. Open WebUI's database and chat data survive pod restarts because they use a PVC. Ollama data is persistent only when you install the optional backend.

**Cleanup:** Delete the namespace to remove Open WebUI, its secret, and its saved chat data. If you installed optional Ollama, this also removes its downloaded models.

```bash
kubectl delete namespace ai
```

## Deploy RAGFlow on Minikube (Community Helm Chart)

> **Learning-only / experimental:** RAGFlow's [upstream deployment guide](https://github.com/infiniflow/ragflow/blob/main/docker/README.md) documents Docker Compose, not this Kubernetes chart. This lab uses the community-maintained [RAGFlow Helm chart](https://github.com/whg517/ragflow-helm-chart), pinned to [commit `78ecb423`](https://github.com/whg517/ragflow-helm-chart/commit/78ecb423416a852857a78b4fcb16e57b23d88bb1). That chart targets RAGFlow `v0.27.0`, not the newer Go-based release line. Review the chart and its dependencies before installing; this is not an upstream-supported or production deployment.

RAGFlow requires several stateful services. This single-node evaluation chart enables PostgreSQL, Valkey, RustFS object storage, and OpenSearch along with RAGFlow. Expect large image downloads and high disk/RAM use. Use a disposable Minikube profile with at least 16 GB host RAM, 6 CPU cores, and 50 GB free disk; allocate 12 GB RAM and 6 CPUs to Minikube. Stop Jenkins, Ollama, and other memory-heavy lab services first. Do not upload sensitive documents or use real production credentials.

Install Helm if it is not already present, and prepare the Minikube node for OpenSearch:

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

The community chart's stock RAGFlow image runs as root; this lab is for a trusted local cluster and does not meet a strict restricted-Pod-Security policy. Do not expose its services to an untrusted network.

`vm.max_map_count` is a host kernel setting required by OpenSearch; the `sysctl` change is temporary and resets after a host reboot. If Helm is already installed, skip the snap command. If the host cannot provide these resources, skip this lab rather than starving other services.

Clone and pin the community chart, then inspect and resolve its Helm dependencies:

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

Create credentials as a Kubernetes Secret. The commands prompt for values so plaintext passwords are not embedded in the chart's values file. Keep the initial admin password in a password manager; do not print the Secret with `kubectl get secret -o yaml`.

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

Configure a deliberately small, single-node evaluation deployment. The in-chart stateful services are not highly available. The chart ships its own values schema; render and inspect the manifests before applying:

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

Install and wait for the chart. The first run downloads multi-gigabyte images and may take a long time:

```bash
echo "Step 5/5: Install RAGFlow and inspect rollout"
helm upgrade --install ragflow . --namespace ragflow \
  --values /tmp/ragflow-minikube-values.yaml \
  --wait --timeout 30m
kubectl get pods,services,pvc -n ragflow -o wide
kubectl get jobs -n ragflow
kubectl rollout status deployment/ragflow-api -n ragflow --timeout=10m
```

If the chart's resource names differ, use `kubectl get deployments,statefulsets,pods -n ragflow` to identify them; the Helm release name prefixes generated resources.

Forward the RAGFlow web service and verify the API:

```bash
kubectl get services -n ragflow
kubectl port-forward -n ragflow service/ragflow-api 8088:80
```

While port-forward is running, open `http://127.0.0.1:8088` in a browser and sign in as `lab-admin@example.invalid` using the password entered above. Stop port-forward with `Ctrl-C`. Configure an LLM and embedding provider in RAGFlow's model settings before creating a knowledge base; the Kubernetes chart does not install a language model. Use a provider/API key intended for the lab and avoid uploading confidential data.

Inspect the chart's workloads and dependencies:

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

For logs, replace `<pod-name>` with a pod from `kubectl get pods -n ragflow`:

```bash
kubectl describe pod -n ragflow <pod-name>
kubectl logs -n ragflow <pod-name> --all-containers=true --tail=200
kubectl logs -n ragflow <pod-name> --all-containers=true --previous --tail=200
```

**Debug:** OpenSearch commonly fails when `vm.max_map_count` is too low; verify it with `sysctl vm.max_map_count`. A pod in `Pending` may indicate insufficient Minikube memory, CPU, or persistent disk. Use events, pod descriptions, and PVC status to identify which dependency is blocked. RAGFlow's readiness health check also verifies its database, cache, document engine, and object storage; wait for all dependencies before diagnosing the API pod. Avoid `helm uninstall` until you have decided whether to preserve the namespace's persistent data.

**Cleanup:** This removes the chart and its in-namespace demo dependencies. Persistent volume reclaim behavior depends on the Minikube StorageClass; inspect PVCs and PVs before deleting them if you need the lab data.

```bash
helm uninstall ragflow -n ragflow
kubectl delete namespace ragflow
```

## Cleanup

Remove the Kubernetes lab resources when finished. Deleting the Jenkins PVC or the `ci` namespace removes Jenkins' stored configuration; export or preserve its data before cleanup if you want to keep the instance.

```bash
echo "Step 1/2: Remove the Node.js application"
kubectl delete -f /tmp/nodejs-demo.yaml --ignore-not-found

echo "Step 2/2: Remove Jenkins and its persistent data"
kubectl delete -f /tmp/jenkins.yaml --ignore-not-found
kubectl delete -f /tmp/jenkins-pvc.yaml --ignore-not-found
kubectl delete namespace ci --ignore-not-found
```

---
