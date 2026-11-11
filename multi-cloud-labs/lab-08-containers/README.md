# Lab 08 – Containerized Application Deployment

Build a small Python HTTP service, package it in Docker, and run it with Compose and a health check. Complete the [WSL and Docker setup](../README.md#start-here).

## 1. Create the application and image definition

```bash
mkdir -p ~/devops-labs/lab08
cd ~/devops-labs/lab08
cat > app.py <<'EOF'
from http.server import BaseHTTPRequestHandler, HTTPServer
import os


class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == "/health":
            body = b"ok\n"
            self.send_response(200)
        else:
            body = f"hello from {os.getenv('APP_ENV', 'local')}\n".encode()
            self.send_response(200)
        self.send_header("Content-Type", "text/plain")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)


HTTPServer(("0.0.0.0", 8080), Handler).serve_forever()
EOF

cat > Dockerfile <<'EOF'
FROM python:3.12-slim
WORKDIR /app
COPY app.py .
USER 10001
EXPOSE 8080
CMD ["python", "app.py"]
EOF

cat > compose.yaml <<'EOF'
services:
  web:
    build: .
    environment:
      APP_ENV: lab
    ports:
      - "127.0.0.1:8088:8080"
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8080/health')"]
      interval: 10s
      timeout: 3s
      retries: 3
EOF
```

## 2. Build, run, and verify

```bash
docker compose config
docker compose up -d --build
docker compose ps
curl --fail-with-body http://127.0.0.1:8088/
curl --fail-with-body http://127.0.0.1:8088/health
docker compose logs --tail 50 web
```

## 3. Stop and remove

```bash
docker compose down
```

Discuss image layers, non-root execution, environment configuration, network exposure, and the difference between a container image registry and a running container service. For managed ECS, Azure Container Apps, or Cloud Run, push only to a lab registry/account after reviewing costs and access controls.
