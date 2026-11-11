# Lab 17 – AI-Powered RAG Application

Run Ollama, Qdrant, and Open WebUI locally, then ingest and query a document through the interface. Model downloads can use several gigabytes of disk space and CPU/RAM. Complete the [WSL and Docker setup](../README.md#start-here).

## 1. Write the Compose stack and a sample document

```bash
mkdir -p ~/devops-labs/lab17
cd ~/devops-labs/lab17
cat > compose.yaml <<'EOF'
services:
  ollama:
    image: ollama/ollama:latest
    ports:
      - "127.0.0.1:11434:11434"
    volumes:
      - ollama_data:/root/.ollama
  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "127.0.0.1:6333:6333"
    volumes:
      - qdrant_data:/qdrant/storage
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    depends_on:
      - ollama
      - qdrant
    ports:
      - "127.0.0.1:3000:8080"
    environment:
      OLLAMA_BASE_URL: http://ollama:11434
      VECTOR_DB: qdrant
      QDRANT_URI: http://qdrant:6333
      RAG_EMBEDDING_ENGINE: ollama
      RAG_EMBEDDING_MODEL: nomic-embed-text
    volumes:
      - open_webui_data:/app/backend/data
volumes:
  ollama_data:
  qdrant_data:
  open_webui_data:
EOF

cat > handbook.txt <<'EOF'
The lab application stores documents in object storage and metadata in a relational database.
An event queue can notify workers when a document is uploaded.
Retrieval-Augmented Generation finds relevant document passages before asking a language model to answer.
EOF
```

## 2. Start services and download local models

```bash
docker compose up -d
docker compose ps
docker compose exec ollama ollama pull llama3.2
docker compose exec ollama ollama pull nomic-embed-text
curl --fail-with-body http://127.0.0.1:6333/healthz
```

## 3. Ingest a document and ask a grounded question

Open `http://127.0.0.1:3000`, create a local Open WebUI account, and select `llama3.2` as the chat model. Create a knowledge collection, upload `handbook.txt`, wait for embedding/indexing to finish, and ask: **“What services does the lab application use, and what triggers a worker?”** Check that the answer is supported by the uploaded text. Also ask a question not covered by the document and note whether the model admits the information is missing.

Qdrant and Ollama are local services; Open WebUI's configuration and screens can change between releases. Do not upload confidential documents or expose these ports outside loopback. A local RAG demo does not validate production isolation, access controls, retention, or model safety.

## 4. Stop the stack

```bash
docker compose down
```

To delete downloaded models, vector data, and WebUI accounts as well:

```bash
docker compose down -v
```
