# HealthLink AI - Deployment Guide

## Project Overview

HealthLink AI is an AI-powered Smart Health Management System built using:

* FastAPI (Backend)
* Streamlit (Frontend)
* Gemini API (LLM & Embeddings)
* Pinecone (Vector Database)
* Docker
* Google Cloud Run
* GitHub Actions CI/CD

---

# Task 1: Run Backend and Frontend Locally

## Prerequisites

* Python 3.13.x
* Docker Desktop
* Git
* Google Cloud SDK
* Pinecone Account
* Gemini API Key

---

## Clone Repository

```bash
git clone https://github.com/<your-github-id>/healthlink-ai.git
cd healthlink-ai
```

---

## Create Virtual Environment

```bash
python -m venv venv
venv\Scripts\activate
```

---

## Install Dependencies

Using UV:

```bash
uv sync
```

OR

```bash
pip install -r requirements.txt
```

---

## Configure Environment Variables

Create `.env`

```env
GEMINI_API_KEY=your_gemini_api_key
PINECONE_API_KEY=your_pinecone_api_key
```

---

## Run Backend

```bash
uv run python main.py
```

Expected:

```text
HealthLink startup complete
Uvicorn running on http://0.0.0.0:8000
```

---

## Verify Backend

```bash
http://localhost:8000/docs
```

```bash
http://localhost:8000/api/v1/health
```

---

## Run Streamlit Frontend

```bash
streamlit run ui/streamlit_app.py
```

Expected:

```text
http://localhost:8501
```

---

# Task 2: Push Code to GitHub Repository

## Initialize Git

```bash
git init
git add .
git commit -m "Initial commit"
```

---

## Create Repository on GitHub

Example:

```text
https://github.com/mishratanushree-ops/healthlink-ai
```

---

## Configure Remote

Check existing remote:

```bash
git remote -v
```

Remove incorrect remote:

```bash
git remote remove origin
```

Add correct remote:

```bash
git remote add origin https://github.com/mishratanushree-ops/healthlink-ai.git
```

Verify:

```bash
git remote -v
```

---

## Push Code

```bash
git branch -M main
git push -u origin main
```

---

# Task 3: Containerize Application Using Docker

## Build Docker Image

```bash
docker build -t healthlink-ai:v1 .
```

Verify:

```bash
docker images
```

---

## Run Docker Container

```bash
docker run -p 8080:8080 ^
-e GEMINI_API_KEY=<your_key> ^
-e PINECONE_API_KEY=<your_key> ^
healthlink-ai:v1
```

Expected:

```text
Application startup complete
```

---

## Verify Running Container

```bash
docker ps
```

Expected:

```text
0.0.0.0:8080->8080/tcp
```

---

## Test API

```bash
curl http://localhost:8080/api/v1/health
```

---

# Task 4: Build CI/CD Pipeline

## Create GitHub Workflow

Create:

```text
.github/workflows/deploy.yml
```

Sample Workflow:

```yaml
name: Deploy to Cloud Run

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - uses: google-github-actions/setup-gcloud@v2

      - run: |
          gcloud builds submit \
            --tag asia-south1-docker.pkg.dev/PROJECT_ID/healthlink-repo/healthlink-ai:${{ github.sha }}

      - run: |
          gcloud run deploy healthlink-ai \
            --image asia-south1-docker.pkg.dev/PROJECT_ID/healthlink-repo/healthlink-ai:${{ github.sha }} \
            --region asia-south1 \
            --allow-unauthenticated
```

---

## GitHub Secrets

Add:

```text
GCP_SA_KEY
GEMINI_API_KEY
PINECONE_API_KEY
```

---

# Task 5: Deploy to GCP

## Login

```bash
gcloud auth login
```

---

## Set Project

```bash
gcloud config set project gen-lang-client-0167173788
```

---

## Enable Services

```bash
gcloud services enable run.googleapis.com
gcloud services enable artifactregistry.googleapis.com
gcloud services enable cloudbuild.googleapis.com
```

---

## Create Artifact Registry

```bash
gcloud artifacts repositories create healthlink-repo \
--repository-format=docker \
--location=asia-south1
```

---

## Configure Docker

```bash
gcloud auth configure-docker asia-south1-docker.pkg.dev
```

---

## Build Image in Cloud Build

(Used because local Docker push had TLS issues)

```bash
gcloud builds submit \
--tag asia-south1-docker.pkg.dev/gen-lang-client-0167173788/healthlink-repo/healthlink-ai:v2
```

---

## Deploy to Cloud Run

```bash
gcloud run deploy healthlink-ai \
--image asia-south1-docker.pkg.dev/gen-lang-client-0167173788/healthlink-repo/healthlink-ai:v2 \
--platform managed \
--region asia-south1 \
--allow-unauthenticated
```

---

## Configure Environment Variables

```bash
gcloud run services update healthlink-ai \
--region asia-south1 \
--set-env-vars GEMINI_API_KEY=<key>,PINECONE_API_KEY=<key>
```

---

## Get Service URL

```bash
gcloud run services describe healthlink-ai \
--region asia-south1 \
--format="value(status.url)"
```

---

## Verify Deployment

Swagger:

```text
https://<cloud-run-url>/docs
```

Health Endpoint:

```text
https://<cloud-run-url>/api/v1/health
```

---

# Troubleshooting Performed

## 1. NumPy Installation Failure

Issue:

```text
Unknown compiler(s): gcc, cl, clang
```

Reason:

* NumPy 1.26.x not compatible with Python 3.13.

Fix:

```text
Upgraded dependencies to:
numpy>=2.2.0
pandas>=2.2.3
```

---

## 2. Pydantic Dependency Conflict

Issue:

```text
ResolutionImpossible
```

Reason:

```text
langchain >=1.x requires pydantic >=2.7.4
```

Fix:

```text
pydantic>=2.7.4
pydantic-settings>=2.7.0
```

---

## 3. UV Package Discovery Error

Issue:

```text
Multiple top-level packages discovered
```

Fix:

Added explicit package discovery in pyproject.toml:

```toml
[tool.setuptools]
packages = ["agents","api","config","core","data","path","ui"]
```

---

## 4. Invalid Gemini API Key

Issue:

```text
ValueError: GEMINI_API_KEY is required
```

Fix:

Configured Cloud Run environment variables.

---

## 5. Cloud Run Startup Failure

Issue:

```text
Container failed to start
```

Reason:

Application was expecting API keys.

Fix:

Added:

```text
GEMINI_API_KEY
PINECONE_API_KEY
```

to Cloud Run service.

---

## 6. Docker Push TLS Error

Issue:

```text
tls: protocol version not supported
```

Reason:

Docker Desktop proxy/TLS issue.

Fix:

Skipped local Docker push and used Cloud Build.

```bash
gcloud builds submit
```

---

## 7. Port Configuration Verification

Checked:

```text
Dockerfile
docker-compose.yml
settings.py
streamlit_app.py
```

Verified:

* Cloud Run container listening on 8080.
* Local development uses 8000.
* Docker maps 8080 correctly.

---

# Final Outcome

✅ Backend running locally

✅ Frontend running locally

✅ Code pushed to GitHub

✅ Docker container created

✅ Artifact Registry image created

✅ Cloud Run deployment successful

✅ Gemini API integrated

✅ Pinecone integrated

✅ CI/CD workflow prepared

✅ Application accessible through GCP Cloud Run
