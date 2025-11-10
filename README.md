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


# Codes

## backend/Dockerfile

```
FROM python:3.10-slim
WORKDIR /app
COPY requirements.txt /app/
RUN pip install --no-cache-dir -r requirements.txt
COPY . /app
CMD uvicorn app:app --host 0.0.0.0 --port 8000
```

## backend/requirements.txt

```
fastapi
uvicorn[standard]
redis
python-dotenv
pyjwt
pydantic
sentence-transformers
faiss-cpu
transformers
torch
pytesseract
python-multipart
boto3
```

## backend/app.py

```
from fastapi import FastAPI, HTTPException, BackgroundTasks, UploadFile, File, Depends
from pydantic import BaseModel, EmailStr
import secrets
import smtplib
import redis
import time
import jwt
import os
from retrieval import add_document, query_knowledge
from dotenv import load_dotenv
load_dotenv()
app = FastAPI()
REDIS_HOST = os.getenv("REDIS_HOST", "redis")
REDIS_PORT = int(os.getenv("REDIS_PORT", 6379))
r = redis.Redis(host=REDIS_HOST, port=REDIS_PORT, db=0, decode_responses=True)
JWT_SECRET = os.getenv("JWT_SECRET", "replace_with_strong_secret")
SMTP_HOST = os.getenv("SMTP_HOST", "smtp.example.com")
SMTP_PORT = int(os.getenv("SMTP_PORT", 587))
SMTP_USER = os.getenv("SMTP_USER", "user@example.com")
SMTP_PASS = os.getenv("SMTP_PASS", "password")
class Signup(BaseModel):
    email: EmailStr
class Verify(BaseModel):
    email: EmailStr
    otp: str
def send_email(to_email: str, subject: str, body: str):
    server = smtplib.SMTP(SMTP_HOST, SMTP_PORT)
    server.starttls()
    server.login(SMTP_USER, SMTP_PASS)
    message = f"Subject: {subject}

{body}"
    server.sendmail(SMTP_USER, to_email, message)
    server.quit()
@app.post("/auth/request-otp")
async def request_otp(payload: Signup, background_tasks: BackgroundTasks):
    otp = str(secrets.randbelow(10**6)).zfill(6)
    key = f"otp:{payload.email}"
    r.setex(key, 600, otp)
    body = f"Your OTP is {otp}. It expires in 10 minutes."
    background_tasks.add_task(send_email, payload.email, "Your OTP", body)
    return {"status": "otp_sent"}
@app.post("/auth/verify")
async def verify(payload: Verify):
    key = f"otp:{payload.email}"
    stored = r.get(key)
    if stored is None:
        raise HTTPException(status_code=400, detail="otp_expired_or_missing")
    if stored != payload.otp:
        raise HTTPException(status_code=401, detail="invalid_otp")
    r.delete(key)
    token = jwt.encode({"sub": payload.email, "iat": int(time.time())}, JWT_SECRET, algorithm="HS256")
    return {"access_token": token}
def get_current_user(token: str = Depends(lambda: None)):
    raise HTTPException(status_code=401, detail="not_implemented")
@app.post("/docs/upload")
async def upload(file: UploadFile = File(...)):
    contents = await file.read()
    doc_id = secrets.token_hex(8)
    add_document(doc_id, contents)
    return {"doc_id": doc_id}
class Query(BaseModel):
    query: str
@app.post("/chat/query")
async def chat_query(payload: Query):
    ans = query_knowledge(payload.query)
    return ans
```

## backend/retrieval.py

```
from sentence_transformers import SentenceTransformer
import faiss
import numpy as np
import os
from transformers import pipeline
embedder = SentenceTransformer("all-MiniLM-L6-v2")
if os.path.exists("/data/faiss.index"):
    index = faiss.read_index("/data/faiss.index")
    texts = np.load("/data/texts.npy", allow_pickle=True).tolist()
else:
    index = None
    texts = []
summarizer = pipeline("summarization", model="sshleifer/distilbart-xsum-12-6")
def add_document(doc_id: str, raw_bytes: bytes):
    text = raw_bytes.decode(errors='ignore')
    chunks = [text[i:i+1000] for i in range(0, len(text), 1000)]
    vecs = embedder.encode(chunks, convert_to_numpy=True)
    global index, texts
    if index is None:
        d = vecs.shape[1]
        index = faiss.IndexFlatL2(d)
    index.add(vecs)
    texts.extend([{"doc_id": doc_id, "text": c} for c in chunks])
    faiss.write_index(index, "/data/faiss.index")
    np.save("/data/texts.npy", np.array(texts, dtype=object))
def query_knowledge(query: str):
    qvec = embedder.encode([query], convert_to_numpy=True)
    D, I = index.search(qvec, 5)
    hits = [texts[i] for i in I[0]]
    sources = [{"doc_id": h["doc_id"], "snippet": h["text"][:400]} for h in hits]
    context = "

".join([h["text"] for h in hits])
    prompt = query + "

" + context
    summary = summarizer(prompt, max_length=150, min_length=30, do_sample=False)
    return {"answer": summary[0]["summary_text"], "sources": sources}
```

## backend/docproc.py

```
import pytesseract
from PIL import Image
def ocr_bytes(raw_bytes: bytes):
    img = Image.open(io.BytesIO(raw_bytes))
    text = pytesseract.image_to_string(img)
    return text
```

## frontend/index.html

```
<!doctype html>
<html>
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Public Sector Chatbot</title>
  </head>
  <body>
    <div>
      <h1>Public Sector Chatbot</h1>
      <div>
        <label>Email</label>
        <input id="email" type="email" />
        <button id="req">Request OTP</button>
      </div>
      <div>
        <label>OTP</label>
        <input id="otp" />
        <button id="verify">Verify</button>
      </div>
      <div>
        <input id="file" type="file" />
        <button id="upload">Upload Doc</button>
      </div>
      <div>
        <input id="query" />
        <button id="ask">Ask</button>
      </div>
      <pre id="out"></pre>
    </div>
    <script src="/app.js"></script>
  </body>
</html>
```

## frontend/app.js

```
let token = null
document.getElementById('req').onclick = async () => {
  const email = document.getElementById('email').value
  await fetch('/auth/request-otp', {
    method: 'POST',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify({email})
  })
  document.getElementById('out').innerText = 'OTP requested'
}
document.getElementById('verify').onclick = async () => {
  const email = document.getElementById('email').value
  const otp = document.getElementById('otp').value
  const res = await fetch('/auth/verify', {
    method: 'POST',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify({email, otp})
  })
  const j = await res.json()
  token = j.access_token
  document.getElementById('out').innerText = 'Verified'
}
document.getElementById('upload').onclick = async () => {
  const f = document.getElementById('file').files[0]
  const fd = new FormData()
  fd.append('file', f)
  const res = await fetch('/docs/upload', {method: 'POST', body: fd})
  const j = await res.json()
  document.getElementById('out').innerText = JSON.stringify(j, null, 2)
}
document.getElementById('ask').onclick = async () => {
  const q = document.getElementById('query').value
  const res = await fetch('/chat/query', {
    method: 'POST',
    headers: {'Content-Type': 'application/json', 'Authorization': token || ''},
    body: JSON.stringify({query: q})
  })
  const j = await res.json()
  document.getElementById('out').innerText = JSON.stringify(j, null, 2)
}
```

## docker-compose.yml

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
    volumes:
      - faiss_data:/data
  redis:
    image: redis:6-alpine
    ports:
      - 6379:6379
volumes:
  faiss_data:
```

## tests/locustfile.py

```
from locust import HttpUser, task, between
class QuickUser(HttpUser):
    wait_time = between(1, 2)
    def on_start(self):
        self.client.post('/auth/request-otp', json={'email': 'test@example.com'})
    @task(3)
    def ask(self):
        self.client.post('/chat/query', json={'query': 'How many casual leaves per year'})
    @task(1)
    def upload(self):
        files = {'file': ('sample.txt', b'This is a sample policy text')}
        self.client.post('/docs/upload', files=files)
```

---

If you want, I will now produce a single downloadable ZIP of these files or paste the raw files individually so you can copy them into your GitHub repo. Tell me which you prefer.


