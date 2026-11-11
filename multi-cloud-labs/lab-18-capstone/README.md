# Lab 18 – Multi-Cloud AI Document Management Platform

## Goal

Design and run a local-first document platform that accepts document text, stores the original in S3-compatible object storage, records metadata in PostgreSQL, creates a local embedding with Ollama, indexes it in Qdrant, and publishes an event to Redis. The services stand in for cloud equivalents; this is not a deployment to AWS, Azure, or Google Cloud.

The sample deliberately keeps its APIs bound to WSL loopback. It uses fake credentials and non-sensitive demo content. Complete labs 01, 06, 08, 10, 13, and 17 first, then complete this integration exercise.

## 1. Create the application

```bash
mkdir -p ~/devops-labs/lab18
cd ~/devops-labs/lab18
cat > requirements.txt <<'EOF'
fastapi
uvicorn[standard]
boto3
psycopg[binary]
redis
requests
EOF

cat > Dockerfile <<'EOF'
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
USER 10001
EXPOSE 8000
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
EOF

cat > app.py <<'EOF'
import json
import os
import re
import uuid

import boto3
import psycopg
import redis
import requests
from botocore.config import Config
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI(title="Local multi-cloud document lab")
db_url = os.environ["DATABASE_URL"]
redis_client = redis.Redis.from_url(os.environ["REDIS_URL"], decode_responses=True)
s3 = boto3.client(
    "s3",
    endpoint_url=os.environ["S3_ENDPOINT"],
    aws_access_key_id=os.environ["S3_ACCESS_KEY"],
    aws_secret_access_key=os.environ["S3_SECRET_KEY"],
    region_name="us-east-1",
    config=Config(signature_version="s3v4"),
)
bucket = "lab18-documents"
qdrant = os.environ["QDRANT_URL"]
ollama = os.environ["OLLAMA_URL"]
collection = "documents"


class Document(BaseModel):
    id: str
    text: str


class Search(BaseModel):
    query: str


def prepare_services():
    with psycopg.connect(db_url) as connection:
        connection.execute(
            "CREATE TABLE IF NOT EXISTS documents "
            "(id text PRIMARY KEY, object_key text NOT NULL, text text NOT NULL)"
        )
    if bucket not in [item["Name"] for item in s3.list_buckets()["Buckets"]]:
        s3.create_bucket(Bucket=bucket)
    requests.put(
        f"{qdrant}/collections/{collection}",
        json={"vectors": {"size": 768, "distance": "Cosine"}},
        timeout=5,
    )


def embed(text):
    response = requests.post(
        f"{ollama}/api/embed",
        json={"model": "nomic-embed-text", "input": text},
        timeout=120,
    )
    response.raise_for_status()
    return response.json()["embeddings"][0]


@app.get("/health")
def health():
    return {"status": "ok"}


@app.post("/documents")
def add_document(document: Document):
    if not re.fullmatch(r"[A-Za-z0-9_-]{1,80}", document.id) or not document.text.strip():
        raise HTTPException(status_code=400, detail="Use a simple document ID and non-empty text")
    try:
        prepare_services()
        key = f"{document.id}.txt"
        s3.put_object(Bucket=bucket, Key=key, Body=document.text.encode(), ContentType="text/plain")
        with psycopg.connect(db_url) as connection:
            connection.execute(
                "INSERT INTO documents VALUES (%s, %s, %s) "
                "ON CONFLICT (id) DO UPDATE SET object_key=EXCLUDED.object_key, text=EXCLUDED.text",
                (document.id, key, document.text),
            )
        vector = embed(document.text)
        requests.put(
            f"{qdrant}/collections/{collection}/points",
            json={"points": [{"id": str(uuid.uuid5(uuid.NAMESPACE_URL, document.id)), "vector": vector, "payload": {"text": document.text}}]},
            timeout=10,
        ).raise_for_status()
        redis_client.lpush("document-events", json.dumps({"event": "DocumentIndexed", "id": document.id}))
        return {"id": document.id, "stored": True, "indexed": True}
    except (requests.RequestException, psycopg.Error, redis.RedisError) as error:
        raise HTTPException(status_code=503, detail=f"Local dependency unavailable: {error}") from error


@app.post("/search")
def search(body: Search):
    try:
        vector = embed(body.query)
        response = requests.post(
            f"{qdrant}/collections/{collection}/points/search",
            json={"vector": vector, "limit": 3, "with_payload": True},
            timeout=10,
        )
        response.raise_for_status()
        return {"matches": response.json()["result"]}
    except requests.RequestException as error:
        raise HTTPException(status_code=503, detail=f"Local search dependency unavailable: {error}") from error
EOF
```

## 2. Start the local service stack

```bash
cat > compose.yaml <<'EOF'
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: lab
      POSTGRES_PASSWORD: local-only-password
      POSTGRES_DB: documents
    ports:
      - "127.0.0.1:5434:5432"
  redis:
    image: redis:7-alpine
    ports:
      - "127.0.0.1:6380:6379"
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: labminio
      MINIO_ROOT_PASSWORD: local-only-password
    ports:
      - "127.0.0.1:9000:9000"
      - "127.0.0.1:9001:9001"
    volumes:
      - minio_data:/data
  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "127.0.0.1:6333:6333"
    volumes:
      - qdrant_data:/qdrant/storage
  ollama:
    image: ollama/ollama:latest
    ports:
      - "127.0.0.1:11434:11434"
    volumes:
      - ollama_data:/root/.ollama
  api:
    build: .
    depends_on:
      - postgres
      - redis
      - minio
      - qdrant
      - ollama
    environment:
      DATABASE_URL: postgresql://lab:local-only-password@postgres:5432/documents
      REDIS_URL: redis://redis:6379/0
      S3_ENDPOINT: http://minio:9000
      S3_ACCESS_KEY: labminio
      S3_SECRET_KEY: local-only-password
      QDRANT_URL: http://qdrant:6333
      OLLAMA_URL: http://ollama:11434
    ports:
      - "127.0.0.1:8000:8000"
volumes:
  minio_data:
  qdrant_data:
  ollama_data:
EOF

docker compose up -d postgres redis minio qdrant ollama
until curl --fail http://127.0.0.1:11434/api/tags; do sleep 2; done
docker compose exec ollama ollama pull nomic-embed-text
docker compose up -d --build api
docker compose ps
until curl --fail http://127.0.0.1:8000/health >/dev/null; do sleep 2; done
until docker compose exec -T postgres pg_isready -U lab -d documents; do sleep 2; done
until docker compose exec -T redis redis-cli ping | grep -q PONG; do sleep 2; done
until curl --fail http://127.0.0.1:6333/healthz >/dev/null; do sleep 2; done
curl --fail http://127.0.0.1:9000/minio/health/ready
```

The initial model download may be several hundred megabytes. Wait until the API container is running before testing requests.

## 3. Upload and search a sample document

```bash
curl --fail-with-body http://127.0.0.1:8000/health
curl --fail-with-body -X POST http://127.0.0.1:8000/documents \
  -H 'Content-Type: application/json' \
  -d '{"id":"cloud-services","text":"Object storage keeps documents. PostgreSQL stores metadata. Redis publishes document events. Ollama generates embeddings and Qdrant indexes vectors for semantic search."}'
curl --fail-with-body -X POST http://127.0.0.1:8000/search \
  -H 'Content-Type: application/json' \
  -d '{"query":"Where are document vectors indexed?"}'
docker compose exec -T postgres psql -U lab -d documents \
  -c 'SELECT id, object_key FROM documents;'
docker compose exec -T redis redis-cli LRANGE document-events 0 -1
```

Inspect the MinIO console at `http://127.0.0.1:9001` (`labminio` / `local-only-password`) and the Qdrant collections at `http://127.0.0.1:6333/collections`. These credentials are for this local exercise only. The example is a learning scaffold, not production-ready: it needs authentication, authorization, asynchronous retries, input-size limits, schema migrations, secret management, observability, and resilient startup/readiness handling.

## 4. Design the cloud mapping and clean up

Map MinIO to S3 / Blob Storage / Cloud Storage; PostgreSQL to RDS / Azure Database for PostgreSQL / Cloud SQL; Redis to provider messaging or cache services; and Qdrant/Ollama to shared AI components. For each cloud, draw the identity boundary, private network, encryption/key ownership, event delivery/retry path, and cost controls. Do not deploy the example to a real account without adding those controls and reviewing the plan.

```bash
docker compose down -v
```

This deletes the local database, object bucket, vectors, and downloaded model data.
