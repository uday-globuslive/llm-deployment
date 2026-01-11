# GCP LLM Deployment - Complete Guide

## Google Cloud Services Landscape

### Quick Service Reference

| Service | Purpose | Use Case |
|---------|---------|----------|
| **Vertex AI** | Managed ML platform | Production ML workloads |
| **Generative AI API** | Pre-trained models | Text, images, code generation |
| **Compute Engine** | Virtual machines | Self-hosted with full control |
| **Cloud Run** | Serverless containers | Simple APIs, auto-scaling |
| **GKE** | Kubernetes (Google managed) | Production scale |
| **Cloud Storage** | Object storage | Model files, configs |
| **Cloud SQL** | Managed databases | Store conversations |
| **Memorystore** | Managed Redis | Cache responses |

---

## Three Ways to Deploy LLMs on GCP

### Option 1: Vertex AI (Recommended for Beginners)

**What it is:** Google's managed ML platform with LLM capabilities

**Supported Models:**
- Gemini Pro (free tier available)
- PaLM 2 (text-bison)
- Embeddings models
- Fine-tune capabilities

**Architecture:**
```
Your Application
    ↓
Vertex AI Python Client
    ↓
Vertex AI (Managed by Google)
    ↓
Response
```

**Pros:**
- ✅ No infrastructure to manage
- ✅ Built-in auto-scaling
- ✅ Free tier available (Gemini Pro)
- ✅ Tight integration with GCP
- ✅ Multimodal (text + images)
- ✅ Fine-tuning support

**Cons:**
- ❌ Not all models available
- ❌ Requires proper setup
- ❌ Limited customization

**Cost Estimate:**
```
Gemini Pro: FREE (within limits)
- 60 requests per minute
- Perfect for learning

PaLM 2 Text-Bison:
- $0.0001 per 1K input tokens
- $0.0003 per 1K output tokens

Example: 1M input, 100K output
Cost = ($0.1) + ($0.03) = $0.13/month
```

**When to use:**
- Prototyping
- Learning AI/ML
- Cost-conscious projects
- Need Google ecosystem integration

**Quick Start Code:**

```python
from vertexai.generative_models import GenerativeModel

model = GenerativeModel("gemini-pro")

response = model.generate_content(
    "What is DevOps?"
)

print(response.text)
```

**Setup Steps:**
1. Create Google Cloud project
2. Enable Vertex AI API
3. Install Google Cloud SDK
4. Set up authentication
5. Write application code

---

### Option 2: Cloud Run (Serverless Containers)

**What it is:** Serverless container platform with auto-scaling

**Supported Models:**
- Any containerized model
- Open-source models
- Custom models

**Architecture:**
```
Application
    ↓
Cloud Run Service (Auto-scales)
    ↓
Container with Model + Framework
    ↓
Cloud Storage (for model files)
```

**Pros:**
- ✅ Serverless (no VM management)
- ✅ Auto-scales to zero (save money)
- ✅ Simple deployment (one command)
- ✅ Pay only for what you use
- ✅ Cold start included in free tier

**Cons:**
- ❌ Cannot use GPU (CPU only)
- ❌ Limited to small models
- ❌ Timeout of 60 minutes
- ❌ High latency for large models

**Cost Estimate:**
```
Small model on Cloud Run (CPU only):
- 1M requests (500ms each):
  - vCPU: 1M × 0.5s / 3600 / 1000 × $0.00002 = $0.01
  - Memory: included in CPU cost
  - Total: ~$10-20/month for 1M requests

No charge if no traffic!
```

**When to use:**
- Small models only (< 7B parameters)
- Development/testing
- Proof of concept
- Low-traffic applications
- Async batch processing

**Quick Setup:**

```bash
# 1. Create Cloud Run service
gcloud run deploy llm-service \
  --source . \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated

# 2. Get service URL
gcloud run services list
```

---

### Option 3: GKE (Kubernetes - Production)

**What it is:** Google's managed Kubernetes service

**Supported Models:**
- Any model in any framework
- Multi-model deployments
- Custom hardware

**Architecture:**
```
Application
    ↓
Load Balancer
    ↓
┌──────────────┬──────────────┬──────────────┐
│ GKE Node     │ GKE Node     │ GKE Node     │
│ (GPU Pod)    │ (GPU Pod)    │ (GPU Pod)    │
└──────────────┴──────────────┴──────────────┘
```

**Pros:**
- ✅ True production-grade
- ✅ Auto-scaling
- ✅ GPU support
- ✅ Multi-region
- ✅ High availability

**Cons:**
- ❌ Complex setup (days)
- ❌ Higher cost at small scale
- ❌ Requires Kubernetes knowledge

**Cost Estimate:**
```
GKE Cluster (3 nodes with GPU):
- Compute (n1-standard-4): $0.19/hour each = $3.21/day
- GPU (Tesla T4): $0.29/hour each = $2.48/day
- 30 days: ~$3,200/month minimum

Break-even with Vertex AI at ~100M tokens/month
```

**When to use:**
- Large-scale production
- Multiple models needed
- Complex architecture
- 99.99% uptime needed

---

## Comparison Table

| Factor | Vertex AI | Cloud Run | GKE |
|--------|-----------|-----------|-----|
| **Setup Time** | Hours | Minutes | Days |
| **Monthly Cost** | Variable | ~$10-50 | $3000+ |
| **GPU Support** | ✅ Yes | ❌ No | ✅ Yes |
| **Auto-scaling** | ✅ Yes | ✅ Yes | ✅ Yes |
| **Multi-model** | Limited | ✅ Yes | ✅ Yes |
| **Customization** | Low | High | Very High |
| **Ops Overhead** | Minimal | Low | High |
| **Free Tier** | Yes (Gemini) | Yes | No |
| **Best For** | Quick start | Development | Production |

---

## Choosing Your Option

### Quick Decision Tree

```
Start here: What's your priority?
    |
    ├─ Cost (want free tier)
    │   └─ Use Vertex AI ✓
    │       (Gemini Pro is free)
    │
    ├─ Speed (want quick deployment)
    │   ├─ Small model needed?
    │   │   ├─ YES → Cloud Run ✓
    │   │   └─ NO → GKE
    │   │
    │   └─ Can wait for setup?
    │       └─ YES → GKE ✓
    │
    └─ Production scale
        └─ Use GKE ✓
            (Best for large scale)
```

---

## Implementation Examples

### Implementation 1: Vertex AI (5 minutes)

```python
# app.py
from vertexai.generative_models import GenerativeModel
from flask import Flask, request, jsonify

app = Flask(__name__)

model = GenerativeModel("gemini-pro")

@app.route('/chat', methods=['POST'])
def chat():
    try:
        data = request.json
        prompt = data.get('prompt', '')
        
        response = model.generate_content(prompt)
        
        return jsonify({
            "response": response.text,
            "model": "gemini-pro"
        })
    except Exception as e:
        return jsonify({"error": str(e)}), 500

if __name__ == '__main__':
    app.run(port=5000)
```

**Setup:**
```bash
pip install google-cloud-aiplatform flask

# Authenticate
gcloud auth application-default login

# Run
python app.py
```

---

### Implementation 2: Cloud Run (10 minutes)

**Dockerfile:**
```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY app.py .

EXPOSE 8080

CMD ["gunicorn", "--bind", "0.0.0.0:8080", "app:app"]
```

**requirements.txt:**
```
Flask==3.0.0
google-cloud-aiplatform==1.40.0
gunicorn==21.2.0
```

**Deploy:**
```bash
# Build and deploy in one command
gcloud run deploy llm-service \
  --source . \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --set-env-vars MODEL_NAME=gemini-pro
```

---

### Implementation 3: GKE (Complex setup)

```bash
# Create GKE cluster
gcloud container clusters create llm-cluster \
  --zone us-central1-a \
  --num-nodes 3 \
  --machine-type n1-standard-4 \
  --enable-ip-alias

# Get credentials
gcloud container clusters get-credentials llm-cluster --zone us-central1-a

# Add GPU node pool
gcloud container node-pools create gpu-pool \
  --cluster llm-cluster \
  --machine-type n1-standard-4 \
  --accelerator type=nvidia-tesla-t4,count=1 \
  --num-nodes 2 \
  --zone us-central1-a

# Deploy with kubectl
kubectl apply -f deployment.yaml
```

---

## Vertex AI Setup (Detailed)

### Step 1: Enable APIs

```bash
# Install gcloud CLI first

# Enable Vertex AI API
gcloud services enable aiplatform.googleapis.com
```

### Step 2: Authentication Setup

```bash
# Method 1: Local development
gcloud auth application-default login

# Method 2: Service account (for production)
gcloud iam service-accounts create llm-service \
  --display-name="LLM Service Account"

gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=serviceAccount:llm-service@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/aiplatform.user
```

### Step 3: Use in Code

```python
import vertexai
from vertexai.generative_models import GenerativeModel

# Initialize Vertex AI
vertexai.init(project="your-project-id", location="us-central1")

# Create model instance
model = GenerativeModel("gemini-pro")

# Generate content
response = model.generate_content("Explain machine learning")
print(response.text)
```

### Step 4: Check Usage

```bash
gcloud logging read "resource.type=api" --limit 10
```

---

## GCP IAM Permissions

### For Vertex AI:
```bash
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=serviceAccount:llm-service@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/aiplatform.user
```

### For Cloud Run:
```bash
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=serviceAccount:llm-service@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/run.invoker
```

### For GKE:
```bash
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=serviceAccount:llm-service@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/container.developer
```

---

## Pricing Comparison

| Service | Cheapest Option | Cost |
|---------|-----------------|------|
| **Vertex AI** | Gemini Pro (free) | $0 (with limits) |
| **Cloud Run** | Small model, low traffic | $10-50/month |
| **GKE** | Minimum cluster | $3000+/month |

---

## Next Steps

1. **Going with Vertex AI?** → See detailed guide (next)
2. **Going with Cloud Run?** → See containerization guide
3. **Going with GKE?** → See Kubernetes guide
4. **Comparing costs?** → Jump to `06-Cost-Optimization/`

---

**Key Takeaway:** GCP offers best free tier (Gemini), best serverless option (Cloud Run), and solid managed Kubernetes (GKE).
