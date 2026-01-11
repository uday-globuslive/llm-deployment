# Azure LLM Deployment - Complete Guide

## Azure Services Landscape

### Quick Service Reference

| Service | Purpose | Use Case |
|---------|---------|----------|
| **Azure OpenAI** | Pre-trained models | Easy, managed, integrated with OpenAI |
| **Azure AI Services** | Pre-trained AI models | Vision, language services |
| **Azure Container Instances (ACI)** | Run containers | Simple containerized deployment |
| **Azure Kubernetes Service (AKS)** | Container orchestration | Production, complex scaling |
| **Azure App Service** | Managed app hosting | Simple web apps, APIs |
| **Azure VMs** | Virtual machines | Full control, self-hosted |
| **Azure Database for PostgreSQL** | Managed database | Store conversations |
| **Azure Cache for Redis** | In-memory cache | Cache responses |
| **Azure Blob Storage** | Object storage | Model files, configs |

---

## Three Ways to Deploy LLMs on Azure

### Option 1: Azure OpenAI Service (Recommended for Beginners)

**What it is:** Azure-hosted OpenAI models with enterprise features

**Supported Models:**
- GPT-4 Turbo
- GPT-4 (including vision)
- GPT-3.5 Turbo
- Embeddings models

**Architecture:**
```
Your Application
    ↓
API Call to Azure OpenAI
    ↓
Azure OpenAI Service (Managed by Microsoft)
    ↓
Response
```

**Pros:**
- ✅ No infrastructure management
- ✅ Built-in auto-scaling
- ✅ Enterprise security features
- ✅ Easy integration with Azure ecosystem
- ✅ Support for function calling, vision
- ✅ Pay per token (no idle cost)

**Cons:**
- ❌ Only OpenAI models available
- ❌ Expensive compared to self-hosted
- ❌ Requires quota request (approval needed)
- ❌ Limited customization

**Cost Estimate:**
```
GPT-4 Turbo:
- Input: $0.01 per 1K tokens
- Output: $0.03 per 1K tokens

GPT-3.5 Turbo (much cheaper):
- Input: $0.0005 per 1K tokens
- Output: $0.0015 per 1K tokens

Example with GPT-3.5: 1M tokens input, 100K output
Cost = (1,000 × $0.0005) + (100 × $0.0015) = $0.65/month
```

**When to use:**
- Prototyping
- Lower cost preferred (GPT-3.5)
- Need vision or advanced models
- Enterprise security required

**Quick Start Code:**

```python
from openai import AzureOpenAI

client = AzureOpenAI(
    api_key="your-api-key",
    api_version="2024-02-15-preview",
    azure_endpoint="https://your-resource.openai.azure.com/"
)

response = client.chat.completions.create(
    model="gpt-35-turbo",
    messages=[
        {"role": "user", "content": "What is DevOps?"}
    ],
    temperature=0.7,
    max_tokens=256
)

print(response.choices[0].message.content)
```

**Setup Steps:**
1. Azure subscription with billing enabled
2. Request Azure OpenAI access (submit form at Azure Portal)
3. Create Azure OpenAI resource
4. Deploy model (GPT-3.5 or GPT-4)
5. Get endpoint and API key
6. Write application code

---

### Option 2: Azure Container Instances (ACI) - Easy Self-Hosted

**What it is:** Simple container deployment without managing Kubernetes

**Supported Models:**
- Llama 2 (any size)
- Mistral
- Any open-source model

**Architecture:**
```
Application
    ↓
Application Gateway (optional)
    ↓
Azure Container Instance
    ↓
Container (with inference framework + model)
```

**Pros:**
- ✅ Simple to deploy (one command)
- ✅ Good for development/testing
- ✅ No Kubernetes complexity
- ✅ Quick to set up (5-10 minutes)
- ✅ Easy to scale (just create more instances)

**Cons:**
- ❌ Limited scaling capabilities
- ❌ Not ideal for production
- ❌ Can't do rolling updates
- ❌ Higher cost per container

**Cost Estimate:**
```
Llama 2 7B in ACI with 1 GPU:
- vCPU: $0.10/hour
- Memory: $0.025/hour per GB
- GPU (NVIDIA A10): $0.88/hour
- Total: ~$1.00/hour = $720/month

Total with storage: ~$750/month
```

**When to use:**
- Development/testing
- Small internal tools
- Proof of concept
- Single instance needed

**Quick Setup:**

```bash
# 1. Create Azure Container Registry
az acr create --resource-group mygroup --name myregistry --sku Basic

# 2. Create Dockerfile
# (See section below)

# 3. Build and push image
az acr build --registry myregistry --image llama2:latest .

# 4. Deploy to ACI
az container create \
  --resource-group mygroup \
  --name llama2-inference \
  --image myregistry.azurecr.io/llama2:latest \
  --cpu 4 --memory 16 \
  --gpu 1 \
  --registry-login-server myregistry.azurecr.io \
  --registry-username username \
  --registry-password password \
  --ports 5000
```

---

### Option 3: Azure Kubernetes Service (AKS) - Production Scale

**What it is:** Managed Kubernetes cluster for production deployments

**Supported Models:**
- Any model you can containerize
- Multiple models in same cluster
- Auto-scaling based on load

**Architecture:**
```
Application
    ↓
Azure Load Balancer
    ↓
┌─────────────┬─────────────┬─────────────┐
│ Kubernetes  │ Kubernetes  │ Kubernetes  │
│ Pod + GPU   │ Pod + GPU   │ Pod + GPU   │
└─────────────┴─────────────┴─────────────┘
    ↓
Shared Persistent Storage
```

**Pros:**
- ✅ True production-grade
- ✅ Auto-scaling
- ✅ Easy updates (rolling deployments)
- ✅ Multi-region support
- ✅ High availability
- ✅ Self-healing

**Cons:**
- ❌ Complex to set up (days)
- ❌ Requires Kubernetes knowledge
- ❌ Higher operational overhead
- ❌ More expensive at small scale

**Cost Estimate:**
```
AKS Cluster (3 nodes, Standard_ND6s with GPU):
- Compute (3 nodes): $3.06 × 3 = $9.18/hour
- Storage (100GB): $5/month
- Ingress: ~$15/month
- Total: ~$6,700/month

Much cheaper than managed services at scale
```

**When to use:**
- Production applications
- High availability required
- Multi-model needed
- 99.9%+ uptime SLA needed

**Quick Setup with Helm:**

```bash
# 1. Create AKS cluster
az aks create \
  --resource-group mygroup \
  --name llm-cluster \
  --node-count 3 \
  --vm-set-type VirtualMachineScaleSets \
  --load-balancer-sku standard

# 2. Get credentials
az aks get-credentials --resource-group mygroup --name llm-cluster

# 3. Add GPU node pool
az aks nodepool add \
  --resource-group mygroup \
  --cluster-name llm-cluster \
  --name gpupool \
  --node-count 2 \
  --vm-set-type VirtualMachineScaleSets \
  --node-vm-size Standard_ND6s_Promo

# 4. Deploy using Helm or kubectl
kubectl apply -f llama2-deployment.yaml
```

---

## Comparison Table: All Three Options

| Factor | Azure OpenAI | ACI | AKS |
|--------|--------------|-----|-----|
| **Setup Time** | Hours | Hours | Days |
| **Monthly Cost (1M tokens)** | $1-20 | $600-1000 | $6000+ |
| **Control Level** | Low | Medium | High |
| **Ops Overhead** | Minimal | Low | High |
| **Auto-scaling** | ✅ Yes | ❌ No | ✅ Yes |
| **Multi-model** | ❌ No | ✅ Yes | ✅ Yes |
| **Custom models** | ❌ No | ✅ Yes | ✅ Yes |
| **High availability** | ✅ Yes | ❌ No | ✅ Yes |
| **Best For** | Quick start | Testing | Production |

---

## Choosing Your Option

### Quick Decision Tree

```
Start here: Do you need production availability?
    |
    ├─ NO → Use Azure OpenAI ✓
    │   (Managed, simple, GPT-4/3.5)
    │
    └─ YES → Do you have Kubernetes expertise?
        |
        ├─ NO → Use ACI ✓
        │   (Simple containers, limited scale)
        │
        └─ YES → Use AKS ✓
            (Production Kubernetes)
```

### Cost-Based Decision

```
Low volume, OpenAI models needed:
→ Azure OpenAI (GPT-3.5 Turbo is cheap)

Dev/test with open-source models:
→ ACI (Simple, good for experimentation)

Production, high volume, multiple models:
→ AKS (Best long-term value)
```

---

## Implementation Examples

### Implementation 1: Azure OpenAI (20 minutes)

```python
# app.py
from openai import AzureOpenAI
from flask import Flask, request, jsonify

app = Flask(__name__)

client = AzureOpenAI(
    api_key="your-api-key",
    api_version="2024-02-15-preview",
    azure_endpoint="https://your-resource.openai.azure.com/"
)

@app.route('/chat', methods=['POST'])
def chat():
    try:
        data = request.json
        prompt = data.get('prompt', '')
        
        response = client.chat.completions.create(
            model="gpt-35-turbo",
            messages=[
                {"role": "user", "content": prompt}
            ],
            temperature=0.7,
            max_tokens=256
        )
        
        return jsonify({
            "response": response.choices[0].message.content,
            "model": "gpt-35-turbo",
            "tokens": response.usage.total_tokens
        })
    except Exception as e:
        return jsonify({"error": str(e)}), 500

if __name__ == '__main__':
    app.run(port=5000)
```

---

### Implementation 2: ACI with Llama 2 (1-2 hours)

**Dockerfile:**
```dockerfile
FROM nvidia/cuda:11.8.0-runtime-ubuntu22.04

RUN apt-get update && apt-get install -y \
    python3 python3-pip git

WORKDIR /app

RUN pip install ollama flask requests

# Copy your inference code
COPY app.py .

EXPOSE 5000

CMD ["python3", "app.py"]
```

**app.py:**
```python
from flask import Flask, request, jsonify
import subprocess

app = Flask(__name__)

@app.route('/chat', methods=['POST'])
def chat():
    data = request.json
    prompt = data.get('prompt', '')
    
    # Ollama runs locally in container
    # Simple HTTP request to local endpoint
    import requests
    
    response = requests.post(
        'http://localhost:11434/api/generate',
        json={
            "model": "llama2",
            "prompt": prompt,
            "stream": False
        }
    )
    
    return jsonify(response.json())

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

**Deployment:**
```bash
# Build and push
az acr build --registry myregistry --image llama2-app:latest .

# Deploy to ACI
az container create \
  --resource-group mygroup \
  --name llama2-inference \
  --image myregistry.azurecr.io/llama2-app:latest \
  --cpu 4 --memory 16 \
  --gpu 1
```

---

### Implementation 3: AKS (Kubernetes deployment)

**kubernetes-deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llama2-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: llama2
  template:
    metadata:
      labels:
        app: llama2
    spec:
      containers:
      - name: llama2
        image: myregistry.azurecr.io/llama2-app:latest
        ports:
        - containerPort: 5000
        resources:
          requests:
            nvidia.com/gpu: 1
          limits:
            nvidia.com/gpu: 1
      nodeSelector:
        accelerator: gpu
---
apiVersion: v1
kind: Service
metadata:
  name: llama2-service
spec:
  selector:
    app: llama2
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5000
  type: LoadBalancer
```

**Deploy:**
```bash
kubectl apply -f kubernetes-deployment.yaml
kubectl get service llama2-service
```

---

## Azure OpenAI Setup Steps (Detailed)

### Step 1: Request Access
1. Azure Portal → Azure OpenAI → Create
2. Choose subscription and resource group
3. Choose region (East US, South Central US, etc.)
4. Choose pricing tier (Standard)
5. Create

**Note:** First-time access requires approval (usually 1-24 hours)

### Step 2: Deploy Model
1. In Azure OpenAI resource → Model deployments
2. Click "Create new deployment"
3. Select model: gpt-35-turbo or gpt-4
4. Choose deployment name: `gpt35`
5. Click Deploy

### Step 3: Get Connection Details
1. Click "Keys and endpoint"
2. Copy:
   - API Key
   - Endpoint (looks like: https://xyz.openai.azure.com/)

### Step 4: Use in Code
```python
from openai import AzureOpenAI

client = AzureOpenAI(
    api_key="YOUR-API-KEY",
    api_version="2024-02-15-preview",
    azure_endpoint="https://YOUR-RESOURCE.openai.azure.com/"
)

response = client.chat.completions.create(
    model="gpt35",  # Your deployment name
    messages=[{"role": "user", "content": "What is Azure?"}]
)

print(response.choices[0].message.content)
```

---

## Azure IAM Permissions

### For Azure OpenAI:
```json
{
  "role": "Cognitive Services OpenAI User",
  "permissions": {
    "actions": [
      "Microsoft.CognitiveServices/*/read",
      "Microsoft.CognitiveServices/accounts/*/actions"
    ]
  }
}
```

### For AKS:
```json
{
  "role": "Azure Kubernetes Service Cluster Admin Role",
  "permissions": {
    "actions": [
      "Microsoft.ContainerService/managedClusters/*"
    ]
  }
}
```

---

## Next Steps

1. **Going with Azure OpenAI?** → See `Azure-OpenAI-Quick-Start.md` (next file)
2. **Going with ACI?** → See `Azure-ACI-Deployment.md`
3. **Going with AKS?** → See `Azure-AKS-Production.md`
4. **Comparing costs?** → Jump to `06-Cost-Optimization/`

---

**Key Takeaway:** Azure OpenAI is easiest to start, ACI good for testing, AKS for production scale.
