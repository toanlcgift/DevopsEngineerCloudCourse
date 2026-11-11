# Lab 10 – Relational Database Deployment

Run PostgreSQL locally, create a database/user/table, connect from the CLI, and test a backup/restore. This does not provision RDS, Azure Database for PostgreSQL, or Cloud SQL. Complete the [WSL and Docker setup](../README.md#start-here).

## 1. Start PostgreSQL

```bash
mkdir -p ~/devops-labs/lab10
cd ~/devops-labs/lab10
cat > compose.yaml <<'EOF'
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: labadmin
      POSTGRES_PASSWORD: local-only-password
      POSTGRES_DB: labdb
    ports:
      - "127.0.0.1:5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U labadmin -d labdb"]
      interval: 5s
      timeout: 3s
      retries: 10
EOF
docker compose up -d
docker compose ps
```

The password is intentionally fake and only for this local disposable database. Do not reuse it.

## 2. Create a least-privilege application role and table

```bash
docker compose exec -T postgres psql -U labadmin -d labdb <<'SQL'
CREATE ROLE app_user LOGIN PASSWORD 'another-local-only-password';
CREATE TABLE orders (
  order_id text PRIMARY KEY,
  status text NOT NULL,
  total numeric(10, 2) NOT NULL CHECK (total >= 0)
);
GRANT SELECT, INSERT, UPDATE ON orders TO app_user;
INSERT INTO orders VALUES ('order-1', 'created', 19.95);
SQL
docker compose exec -T postgres psql -U app_user -d labdb \
  -c "SELECT order_id, status, total FROM orders;"
```

Try a `DELETE` as `app_user` and observe that it is denied. Discuss network boundaries, TLS, backups, maintenance windows, and managed database responsibilities.

## 3. Back up, restore, and clean up

```bash
docker compose exec -T postgres pg_dump -U labadmin -d labdb \
  --format=custom > /tmp/lab10-backup.dump
docker compose exec -T postgres createdb -U labadmin restore_check
docker compose exec -T postgres pg_restore -U labadmin -d restore_check \
  --no-owner < /tmp/lab10-backup.dump
docker compose exec -T postgres psql -U labadmin -d restore_check \
  -c 'SELECT count(*) FROM orders;'
docker compose down -v
rm -f /tmp/lab10-backup.dump
```

`docker compose down -v` permanently deletes this lab database volume.
