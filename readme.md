# 🚀 GCP Cloud Run CI/CD Project (GitHub → Artifact Registry → Cloud Run)

This project demonstrates a complete **CI/CD workflow on Google Cloud Platform (GCP)** where a simple Flask application is:

* Stored in **GitHub**
* Containerized using **Docker**
* Built & pushed to **Artifact Registry**
* Deployed on **Cloud Run**
* Served publicly on **Port 80**

---

## 🧱 Architecture Overview

```
GitHub Repository
      ↓
GitHub Actions (CI)
      ↓
Docker Image
      ↓
Artifact Registry (GCP)
      ↓
Cloud Run
      ↓
Public HTTPS Endpoint (Port 80)
```

---

## ✅ Prerequisites

* Google Cloud account
* GCP Project created
* `gcloud` CLI installed
* GitHub account

---

## 2️⃣ Enable Required GCP APIs

Enable these APIs **once per project**:

```bash
gcloud services enable \
  run.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  iam.googleapis.com \
  compute.googleapis.com
```

⚠️ These APIs are mandatory for Cloud Run and Artifact Registry.

---

## 3️⃣ Install & Configure gcloud

### Check gcloud installation

```bash
gcloud version
```

### Authenticate

```bash
gcloud auth login
gcloud auth application-default login
```

### Set GCP project

```bash
gcloud config set project YOUR_PROJECT_ID
```

---

# 🚀 PHASE 1 – GitHub Repository & Application Setup

---

## 1️⃣ Create GitHub Repository

* **Repository name:** `gcp-cloudrun-cicd`
* **Visibility:** Public or Private

---

## 2️⃣ Create Simple Flask Application (Port 80)

### `app.py`

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello from GCP Cloud Run 🚀"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=80)
```

### `requirements.txt`

```
flask
```

---

## 3️⃣ Dockerfile (VERY IMPORTANT)

### `Dockerfile`

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY app.py .

EXPOSE 80

CMD ["python", "app.py"]
```

### Important Notes

* Cloud Run requires the application to listen on the provided port
* `0.0.0.0` is mandatory for container networking
* Port **80** is explicitly used

---

## 4️⃣ Push Code to GitHub

```bash
git init
git add .
git commit -m "Initial Flask app with Dockerfile"
git branch -M main
git remote add origin https://github.com/<your-username>/gcp-cloudrun-cicd.git
git push -u origin main
```

---

## 📁 Project Structure

```
gcp-cloudrun-cicd/
│
├── app.py
├── requirements.txt
├── Dockerfile
└── README.md
```

##

---

# 🧭 PHASE 2 (GUI) – Artifact Registry + GitHub → GCP Authentication

## 🧱 Visual Flow (Keep This in Mind)

```
GitHub Actions
   ↓ (OIDC)
Workload Identity Provider (GCP IAM)
   ↓
Service Account
   ↓
Artifact Registry
```

---

## 1️⃣ Create Artifact Registry (GCP Console)

### 🔎 Navigation

```
GCP Console → Artifact Registry → Repositories → Create Repository
```

### 📝 Repository Configuration

Fill the form with the following values:

* **Name:** `cloudrun-repo`
* **Format:** Docker
* **Mode:** Standard
* **Region:** `asia-south1`
* **Encryption:** Google-managed key

Click **Create**.

✅ You now have a **private Docker Artifact Registry**.

---

### 🔍 IMPORTANT – Copy Repository Path

After creation:

1. Click on the repository
2. Copy the **Repository path**

It will look like:

```
asia-south1-docker.pkg.dev/PROJECT_ID/cloudrun-repo
```

📌 This value will be used inside **GitHub Actions** to push Docker images.

---

## 2️⃣ Create Service Account (GCP Console)

### 🔎 Navigation

```
IAM & Admin → Service Accounts → Create Service Account
```

---

### Step 1: Service Account Details

* **Name:** `github-actions-sa`
* **Description:** GitHub Actions CI/CD

Click **Create and Continue**.

---

### Step 2: Grant Required Roles

Add the following roles **one by one**:

* **Artifact Registry Writer**
* **Cloud Run Admin**
* **Service Account User**

Click **Done**.

---

## 🔐 What is **Workload Identity**?

**Workload Identity** is a **secure way for applications (workloads) to authenticate to Google Cloud without using long-lived service account keys**.

## 3️⃣ Create Workload Identity Pool (Console)

### Navigation

**IAM & Admin → Workload Identity Federation → Create Pool**

### Pool Details

* **Name**: `github-pool`
* **Description**: GitHub Actions Pool
* **Location**: Global

Click **Continue**

## 4️⃣ Create Workload Identity Provider (GitHub)

### Provider Settings

* **Provider type**: OpenID Connect (OIDC)
* **Provider name**: `github-provider`
* **Issuer URL**:

```
https://token.actions.githubusercontent.com

```

Click **Continue**

---

### Attribute Mapping (VERY IMPORTANT)

In **Attribute Mapping** section:

| Google attributeAssertion |                        |
| ------------------------- | ---------------------- |
| `google.subject`          | `assertion.sub`        |
| `attribute.repository`    | `assertion.repository` |

Click **Save**

✅ GCP now trusts GitHub tokens.

---

## 5️⃣ Link GitHub Repo to Service Account (Console)

This is the **most critical security step**.

### Navigation

**IAM & Admin → Service Accounts → github-actions-sa → Permissions → Grant Access**

### New Principal

Paste this (replace values):

principalSet://iam.googleapis.com/projects/367605285780/locations/global/workloadIdentityPools/github-pool/attribute.repository/Devansh-Chauhan-GitHub/gcp-cloudrun-cicd

---

