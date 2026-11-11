# Lab 09 – Kubernetes Application Deployment

Deploy a local application on Minikube, configure a Deployment, Service, ConfigMap, and Secret, then scale and update it. Complete Docker setup in the [root guide](../README.md#start-here).

## 1. Install kubectl and Minikube

```bash
echo "Step 1/2: Install kubectl"
curl -fsSLo /tmp/kubectl "https://dl.k8s.io/release/$(curl -fsSL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -m 0755 /tmp/kubectl /usr/local/bin/kubectl
rm /tmp/kubectl
echo "Step 2/2: Install Minikube"
curl -fsSLo /tmp/minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install -m 0755 /tmp/minikube /usr/local/bin/minikube
rm /tmp/minikube
minikube version
kubectl version --client
```

## 2. Start a local cluster and write manifests

```bash
minikube start --driver=docker
kubectl create namespace lab09
kubectl -n lab09 create configmap web-config --from-literal=APP_ENV=lab
kubectl -n lab09 create secret generic demo-secret --from-literal=DEMO_TOKEN=not-a-real-token
mkdir -p ~/devops-labs/lab09
cd ~/devops-labs/lab09
cat > app.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: lab09
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx:1.27
          ports:
            - containerPort: 80
          envFrom:
            - configMapRef:
                name: web-config
            - secretRef:
                name: demo-secret
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 2
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: lab09
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
EOF
```

The Secret above is only a Kubernetes API demonstration; base64 encoding is not encryption. Do not use a real credential.

## 3. Deploy, scale, and verify a rollout

```bash
kubectl apply -f app.yaml
kubectl -n lab09 rollout status deployment/web
kubectl -n lab09 get deployments,pods,services
kubectl -n lab09 scale deployment/web --replicas=3
kubectl -n lab09 rollout status deployment/web
kubectl -n lab09 set image deployment/web nginx=nginx:1.27-alpine
kubectl -n lab09 rollout status deployment/web
kubectl -n lab09 port-forward service/web 8089:80
```

In another terminal, run `curl --fail-with-body http://127.0.0.1:8089/`; stop the port-forward with `Ctrl-C`.

## 4. Remove resources

```bash
kubectl delete namespace lab09
minikube delete
```

Ingress and persistent volumes are omitted to keep the first local cluster exercise small. Explore `minikube addons enable ingress` and local PV behavior as an extension. Minikube success does not imply EKS, AKS, or GKE production readiness.
