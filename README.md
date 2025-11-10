# AI-driven_Chatbot

A prototype chatbot for a large public sector organization that answers HR, IT, events, and other internal queries. This repo contains a minimal, hackathon-ready skeleton to demonstrate retrieval-augmented generation, document processing (OCR + layout parsing + summarization), email 2FA, and a lightweight API. Drop it into GitHub, fork it, run it locally, and flex on the judges.

## TL;DR

Spin up the Docker Compose stack, open the frontend, upload a sample HR or IT PDF, then ask the chatbot questions. The system will index the document, create embeddings, and answer grounded queries with source citations. Email-based 2FA is included for demo authentication.

## Features

* Retrieval-Augmented Generation (RAG) demo with sentence-transformers and FAISS
* Document processing pipeline: OCR, chunking, embedding, summarization
* FastAPI backend with async endpoints for chat, document upload, and auth (email OTP)
* Simple web UI to interact with the bot and upload documents (optional)
* Docker Compose for local demo
* Minimal instrumentation and test script for simulating 5 concurrent users

## Architecture

Client -> API (FastAPI) -> Retrieval service (FAISS) -> LLM

Document upload -> OCR -> chunk -> embeddings -> vector DB

Auth: Email OTP stored temporarily in Redis -> JWT session

## Repo layout

```
publicsector-chatbot/
  README.md
  docker-compose.yml
  backend/
    app.py
    auth.py
    retrieval.py
    docproc.py
    requirements.txt
  frontend/
    index.html
    app.js
  sample_docs/
    hr_policy_sample.pdf
    it_guide_sample.pdf
  tests/
    locustfile.py
  LICENSE
```

## Quickstart (requirements)

* Docker and Docker Compose (version 1.29+)
* Python 3.10+ (only if running services locally without Docker)
* An SMTP account for OTP email or local SMTP dev server

## Environment variables

Set these before running or place them in a .env file consumed by docker-compose

```
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USER=you@example.com
SMTP_PASS=securepassword
JWT_SECRET=replace_with_a_strong_secret
REDIS_HOST=redis
FAISS_INDEX_PATH=/data/faiss.index
S3_ENDPOINT=http://minio:9000
S3_ACCESS_KEY=minio
S3_SECRET_KEY=minio123
```

## Docker Compose (local demo)

```
version: '3.8'
services:
  backend:
    build: ./backend
    ports:
      - 8000:8000
    environment:
      - SMTP_HOST=${SMTP_HOST}
      - SMTP_PORT=${SMTP_PORT}
      - SMTP_USER=${SMTP_USER}
      - SMTP_PASS=${SMTP_PASS}
      - JWT_SECRET=${JWT_SECRET}
      - REDIS_HOST=redis
    depends_on:
      - redis
      - faiss
    command: uvicorn app:app --host 0.0.0.0 --port 8000
  redis:
    image: redis:6-alpine
    ports:
      - 6379:6379
  faiss:
    image: ghcr.io/some/faiss-demo:latest
    volumes:
      - faiss_data:/data
  minio:
    image: minio/minio
    command: server /data
    environment:
      - MINIO_ROOT_USER=minio
      - MINIO_ROOT_PASSWORD=minio123
    ports:
      - 9000:9000
volumes:
  faiss_data:
```

## Backend endpoints (examples)

```
POST /auth/request-otp
body { "email": "user@example.com" }

POST /auth/verify
body { "email": "user@example.com", "otp": "123456" }

POST /docs/upload
headers Authorization: Bearer <token>
form file=@hr_policy_sample.pdf

POST /chat/query
headers Authorization: Bearer <token>
body { "query": "How many casual leaves am I allowed?" }
```


## Document processing notes

* OCR stage uses Tesseract for the hackathon. For better accuracy, swap for a cloud OCR provider.
* Chunking: split text into 400-token chunks with 50-token overlap.
* Embeddings: all-MiniLM-L6-v2 from sentence-transformers for low-latency demo.
* Vector DB: FAISS local index for nearest neighbors.


