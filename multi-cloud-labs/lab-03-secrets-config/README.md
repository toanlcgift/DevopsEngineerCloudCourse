# Lab 03 – Secrets and Configuration Management

Run OpenBao locally as a secrets manager, store a secret in its KV v2 engine, and retrieve it without writing the secret into the project. OpenBao is a learning substitute, not AWS Secrets Manager, Azure Key Vault, or Google Secret Manager. Complete the [WSL and Docker setup](../README.md#start-here).

## 1. Start OpenBao in development mode

Development mode uses a known in-memory token and is strictly for disposable local exercises:

```bash
docker run -d --name lab03-openbao \
  -p 127.0.0.1:8200:8200 \
  -e BAO_DEV_ROOT_TOKEN_ID=lab-root-token \
  quay.io/openbao/openbao:latest server -dev
docker logs --tail 30 lab03-openbao
export BAO_ADDR=http://127.0.0.1:8200
export BAO_TOKEN=lab-root-token
```

## 2. Store and retrieve a sample secret

Create a fake database password, enable KV v2, and read the value back. Keep the token and sample secret in the current terminal only.

```bash
docker exec -e BAO_ADDR=http://127.0.0.1:8200 -e BAO_TOKEN=lab-root-token \
  lab03-openbao bao kv put secret/lab03/database username=labuser password=not-a-real-password
docker exec -e BAO_ADDR=http://127.0.0.1:8200 -e BAO_TOKEN=lab-root-token \
  lab03-openbao bao kv get secret/lab03/database
```

## 3. Define a read-only policy

```bash
cat > /tmp/lab03-read.hcl <<'EOF'
path "secret/data/lab03/*" {
  capabilities = ["read"]
}
EOF
docker cp /tmp/lab03-read.hcl lab03-openbao:/tmp/read.hcl
docker exec -e BAO_ADDR=http://127.0.0.1:8200 -e BAO_TOKEN=lab-root-token \
  lab03-openbao bao policy write lab03-read /tmp/read.hcl
docker exec -e BAO_ADDR=http://127.0.0.1:8200 -e BAO_TOKEN=lab-root-token \
  lab03-openbao bao policy read lab03-read
READ_TOKEN=$(docker exec -e BAO_ADDR=http://127.0.0.1:8200 \
  -e BAO_TOKEN=lab-root-token lab03-openbao \
  bao token create -field=token -policy=lab03-read -orphan)
docker exec -e BAO_ADDR=http://127.0.0.1:8200 -e BAO_TOKEN="$READ_TOKEN" \
  lab03-openbao bao kv get secret/lab03/database
if docker exec -e BAO_ADDR=http://127.0.0.1:8200 -e BAO_TOKEN="$READ_TOKEN" \
  lab03-openbao bao kv put secret/lab03/database password=should-be-denied \
  2>/tmp/lab03-denied.log; then
  echo "ERROR: the read-only policy unexpectedly allowed a write"
  exit 1
elif grep -Eiq 'permission denied|not authorized|permission to perform' /tmp/lab03-denied.log; then
  echo "Expected: the read-only token cannot write the secret"
else
  cat /tmp/lab03-denied.log
  exit 1
fi
```

Discuss why a production workload should receive a short-lived identity and a narrowly scoped policy, rather than a shared root token. Avoid storing secrets in `.env` files committed to Git, command history, logs, or container images.

## 4. Clean up

Development data is in the container; deleting the container removes it:

```bash
docker rm -f lab03-openbao
rm -f /tmp/lab03-read.hcl
rm -f /tmp/lab03-denied.log
unset BAO_ADDR BAO_TOKEN
unset READ_TOKEN
```
