# Lab 16 – Event-Driven Data Engineering Pipeline

Process a small CSV locally, store normalized rows in PostgreSQL, and publish processing events to Redis. This demonstrates the pipeline shape without creating cloud storage, function, queue, or database resources. Complete the [WSL and Docker setup](../README.md#start-here).

## 1. Start local PostgreSQL and Redis

```bash
mkdir -p ~/devops-labs/lab16
cd ~/devops-labs/lab16
cat > compose.yaml <<'EOF'
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: lab
      POSTGRES_PASSWORD: local-only-password
      POSTGRES_DB: pipeline
    ports:
      - "127.0.0.1:5433:5432"
  redis:
    image: redis:7-alpine
    ports:
      - "127.0.0.1:6379:6379"
EOF
docker compose up -d
docker compose ps
until docker compose exec -T postgres pg_isready -U lab -d pipeline; do sleep 2; done
until docker compose exec -T redis redis-cli ping | grep -q PONG; do sleep 2; done
```

## 2. Write sample input and a processor

```bash
cat > orders.csv <<'EOF'
order_id,customer,total
o-100,alice,19.95
o-101,bob,32.50
EOF

cat > process.py <<'EOF'
import csv
import json
import os
from decimal import Decimal

import psycopg
import redis

connection = psycopg.connect(os.environ["DATABASE_URL"])
connection.execute(
    "CREATE TABLE IF NOT EXISTS processed_orders "
    "(order_id text PRIMARY KEY, customer text NOT NULL, total numeric(10,2) NOT NULL)"
)
queue = redis.Redis.from_url(os.environ["REDIS_URL"], decode_responses=True)

with open("orders.csv", newline="", encoding="utf-8") as source:
    for row in csv.DictReader(source):
        order = {
            "order_id": row["order_id"],
            "customer": row["customer"],
            "total": str(Decimal(row["total"])),
        }
        connection.execute(
            "INSERT INTO processed_orders VALUES (%s, %s, %s) "
            "ON CONFLICT (order_id) DO UPDATE SET customer=EXCLUDED.customer, total=EXCLUDED.total",
            (order["order_id"], order["customer"], order["total"]),
        )
        queue.lpush("lab16:events", json.dumps({"event": "OrderProcessed", **order}))

connection.commit()
print("Processed input rows and published events to lab16:events")
EOF
```

## 3. Run and verify the pipeline

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install "psycopg[binary]" redis
export DATABASE_URL=postgresql://lab:local-only-password@127.0.0.1:5433/pipeline
export REDIS_URL=redis://127.0.0.1:6379/0
python process.py
docker compose exec -T postgres psql -U lab -d pipeline \
  -c 'SELECT order_id, customer, total FROM processed_orders ORDER BY order_id;'
docker compose exec -T redis redis-cli LRANGE lab16:events 0 -1
```

Run `python process.py` again. The database upsert is repeatable, but this simple producer publishes duplicate events; discuss idempotency, transactional outbox patterns, retries, and dead-letter handling. In a cloud implementation, object-created events can trigger serverless processing and a managed queue, but their delivery and IAM policies need provider-specific testing.

## 4. Clean up

```bash
docker compose down -v
deactivate
rm -rf ~/devops-labs/lab16
unset DATABASE_URL REDIS_URL
```
