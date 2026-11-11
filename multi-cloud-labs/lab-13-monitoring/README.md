# Lab 13 – Monitoring and Observability

Run Prometheus and Grafana locally, scrape Prometheus's own metrics, and explore targets and dashboards. This lab does not connect to a real AWS, Azure, or Google Cloud monitoring account. Complete the [WSL and Docker setup](../README.md#start-here).

## 1. Write Prometheus and Grafana configuration

```bash
mkdir -p ~/devops-labs/lab13/grafana/provisioning/datasources
cd ~/devops-labs/lab13
cat > prometheus.yml <<'EOF'
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["prometheus:9090"]
EOF

cat > grafana/provisioning/datasources/prometheus.yml <<'EOF'
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
EOF

cat > compose.yaml <<'EOF'
services:
  prometheus:
    image: prom/prometheus:v2.55.1
    command: ["--config.file=/etc/prometheus/prometheus.yml"]
    ports:
      - "127.0.0.1:9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
  grafana:
    image: grafana/grafana:11.3.0
    ports:
      - "127.0.0.1:3000:3000"
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: lab-only-password
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
EOF
```

## 2. Start and inspect metrics

```bash
docker compose up -d
docker compose ps
curl --fail-with-body http://127.0.0.1:9090/-/healthy
curl --fail-with-body http://127.0.0.1:9090/api/v1/targets
```

Open Prometheus at `http://127.0.0.1:9090/targets`. Open Grafana at `http://127.0.0.1:3000` and sign in with `admin` / `lab-only-password`; change the password if you keep this instance running. In **Explore**, query `up` and `prometheus_http_requests_total`.

## 3. Examine logs and clean up

```bash
docker compose logs --tail 100 prometheus grafana
docker compose down -v
```

Add a dashboard panel for `up`, then simulate an unavailable target by changing the target name in `prometheus.yml`, restarting Prometheus, and checking the target error. The local sample password and configuration must not be reused outside this disposable lab.
