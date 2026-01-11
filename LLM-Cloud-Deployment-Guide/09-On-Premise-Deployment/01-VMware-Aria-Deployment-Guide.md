# On-Premise LLM Deployment with VMware Aria

## Overview

This guide covers deploying LLMs on-premise using VMware infrastructure, which is essential for regulated industries and data-sensitive applications.

---

## Table of Contents

1. [Infrastructure Requirements](#infrastructure-requirements)
2. [VMware Aria Setup](#vmware-aria-setup)
3. [Kubernetes Deployment](#kubernetes-deployment)
4. [Model Serving with vLLM](#model-serving-with-vllm)
5. [Monitoring and Management](#monitoring-and-management)
6. [Cost Analysis](#cost-analysis)
7. [Security & Compliance](#security--compliance)
8. [Disaster Recovery](#disaster-recovery)
9. [Troubleshooting](#troubleshooting)

---

## Infrastructure Requirements

### Hardware Specifications

**Compute Nodes:**
```
Per Node:
- CPU: 16+ cores (Intel Xeon or AMD EPYC)
- RAM: 128GB minimum
- Storage: NVMe SSD 500GB+ (for OS/applications)
- Network: 10Gbps+ connection

Recommended Configuration:
- 4 nodes minimum (1 control plane + 3 workers)
- Each node: $15,000-20,000
```

**GPU Requirements:**
```
For LLM Inference:
- NVIDIA A100 (80GB): Best performance
  - Cost: $10,000 per card
  - Throughput: 312 TFLOPS
  - Memory: 80GB HBM2

- NVIDIA A10 (24GB): Mid-range option
  - Cost: $4,000 per card
  - Throughput: 156 TFLOPS
  - Memory: 24GB GDDR6

- NVIDIA L40S (48GB): Good balance
  - Cost: $7,000 per card
  - Throughput: 362 TFLOPS
  - Memory: 48GB GDDR6

Recommended for 1M+ inferences/month:
- 4x A100 GPUs = $40,000
- Total bandwidth: 1.2 TB/s
```

**Storage:**
```
Model Storage:
- Llama 2 7B: 15GB
- Llama 2 70B: 140GB
- Mistral 7B: 15GB
- Recommended: 500GB+ SAS or NVMe

Recommended Setup:
- Primary: 500GB SSD for model cache
- Secondary: 2TB RAID 6 for backups
```

**Network:**
```
Bandwidth Requirements:
- Per node: 10Gbps minimum
- Cluster interconnect: 100Gbps fabric recommended
- Storage network: 10Gbps dedicated (NFS/iSCSI)

Architecture:
- Separate VLAN for LLM workloads
- Direct attach GPU <-> CPU (no switch hops)
- Low-latency scheduling preferred (<1ms)
```

### Data Center Requirements

```
Physical Space:
- Power: 20-30 kW for 4-node cluster with 4 GPUs
- Cooling: Adequate for heat dissipation
- Redundancy: Dual power supplies, UPS
- Network redundancy: Multiple ISP connections (optional)

Environmental:
- Temperature: 15-25°C optimal
- Humidity: 40-60% RH
- UPS: 4-8 hours minimum for graceful shutdown
```

---

## VMware Aria Setup

### Prerequisites

```
1. VMware vSphere 7.0+ deployed
2. vSAN or external storage (iSCSI/NFS)
3. VMware NSX for networking (optional but recommended)
4. VMware Aria Automation installed
5. NVIDIA GPU drivers installed in ESXi
```

### Step 1: Enable GPU Support in vSphere

```bash
# SSH to ESXi host
ssh root@esxi-host

# Enable GPU passthrough in BIOS/UEFI (manual step on host)
# Then configure vSphere:

# 1. Create resource pool for GPU workloads
esxcli system module parameters set -m nvidia -p vGPU=1

# 2. Configure shared passthrough GPU
# Via vSphere Client:
# - Edit host settings
# - Hardware > PCI Devices
# - Select GPU > Edit
# - Check "Share GPU passthrough with other workloads"

# 3. Verify GPU availability
nvidia-smi
lspci | grep NVIDIA

# Output should show:
# 01:00.0 3D controller: NVIDIA Corporation GA100 [A100 PCI-e] (rev a1)
```

### Step 2: Create VMware Aria Automation Project

```bash
# Via Aria CLI or Web UI
aria project create \
  --name "llm-inference" \
  --description "On-premise LLM serving infrastructure" \
  --cloud vmware

# Configure cloud account
aria cloud-account create \
  --name vmware-dc \
  --type vmware \
  --hostname vcenter.company.com \
  --username administrator@vsphere.local \
  --password "secure-password"
```

### Step 3: Deploy Kubernetes via Aria

```bash
# Method 1: Using Tanzu Kubernetes Grid (TKG)
# TKG provides enterprise-grade Kubernetes on vSphere

aria kubernetes cluster create \
  --name llm-k8s-cluster \
  --version 1.27 \
  --nodes 4 \
  --worker-cpu 16 \
  --worker-memory 128 \
  --storage-class vmware-storage \
  --network-name llm-workload \
  --cloud-account vmware-dc

# This creates:
# - 1 control plane node
# - 3 worker nodes
# - High availability etcd
# - Integrated storage
```

### Step 4: Configure GPU Support in Kubernetes

```bash
# Connect to Kubernetes cluster
kubectl config use-context llm-k8s-cluster

# Install NVIDIA GPU Operator
# This automatically installs drivers, runtime, and device plugins

helm repo add nvidia https://nvidia.github.io/gpu-operator
helm repo update

helm install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator-system \
  --create-namespace \
  --set driver.enabled=true \
  --set toolkit.enabled=true \
  --set operator.defaultRuntime=containerd

# Verify GPU is available
kubectl get nodes -o wide
kubectl get pods -n gpu-operator-system

# Test GPU access
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: gpu-test
spec:
  containers:
  - name: cuda-test
    image: nvidia/cuda:12.0.0-runtime-ubuntu22.04
    command: ["nvidia-smi"]
    resources:
      limits:
        nvidia.com/gpu: 1
EOF

# Check logs
kubectl logs gpu-test
```

---

## Kubernetes Deployment

### Step 1: Create LLM Serving Namespace

```bash
kubectl create namespace llm-serving
kubectl label namespace llm-serving workload-type=ai
```

### Step 2: Set up Model Storage (PersistentVolume)

```yaml
# File: pvc-model-storage.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: model-storage
  namespace: llm-serving
spec:
  accessModes:
    - ReadWriteMany  # Required for multiple pods
  storageClassName: vmware-storage
  resources:
    requests:
      storage: 500Gi

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cache-storage
  namespace: llm-serving
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: vmware-storage
  resources:
    requests:
      storage: 100Gi
```

**Deploy:**
```bash
kubectl apply -f pvc-model-storage.yaml
kubectl get pvc -n llm-serving
```

### Step 3: Download Models to Storage

```python
# File: download_models.py
import os
from huggingface_hub import snapshot_download

# Models to download
MODELS = [
    "meta-llama/Llama-2-7b-hf",
    "meta-llama/Llama-2-70b-hf",
    "mistralai/Mistral-7B-v0.1"
]

# Mount point from Kubernetes PVC
MODEL_CACHE = "/mnt/models"

os.makedirs(MODEL_CACHE, exist_ok=True)

for model_name in MODELS:
    print(f"Downloading {model_name}...")
    
    snapshot_download(
        repo_id=model_name,
        cache_dir=MODEL_CACHE,
        local_files_only=False,
        resume_download=True,
        force_download=False
    )
    
    print(f"✓ {model_name} ready")

print("\nAll models cached and ready for serving!")
```

**Run as one-time pod:**
```bash
# Create pod to download models
kubectl run model-downloader \
  --image python:3.11 \
  --volume-mount model-storage:/mnt/models \
  -- bash -c "pip install huggingface-hub && python download_models.py"

# Monitor progress
kubectl logs -f model-downloader
```

### Step 4: Deploy vLLM Server

```yaml
# File: vllm-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-server
  namespace: llm-serving
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vllm
  
  template:
    metadata:
      labels:
        app: vllm
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8000"
    
    spec:
      # Use GPU nodes
      nodeSelector:
        kubernetes.io/hostname: worker-node-with-gpu
      
      # GPU resource affinity
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - vllm
              topologyKey: kubernetes.io/hostname
      
      containers:
      - name: vllm
        image: vllm/vllm-openai:v0.2.0
        imagePullPolicy: IfNotPresent
        
        # Environment
        env:
        - name: MODEL_NAME
          value: "meta-llama/Llama-2-7b-chat-hf"
        
        - name: TENSOR_PARALLEL_SIZE
          value: "1"
        
        - name: GPU_MEMORY_UTILIZATION
          value: "0.9"
        
        - name: ENABLE_PREFIX_CACHING
          value: "true"
        
        - name: DISABLE_LOG_STATS
          value: "false"
        
        # Ports
        ports:
        - name: http
          containerPort: 8000
        - name: metrics
          containerPort: 8001
        
        # Resources
        resources:
          requests:
            memory: "32Gi"
            cpu: "8"
            nvidia.com/gpu: "1"
          limits:
            memory: "64Gi"
            cpu: "16"
            nvidia.com/gpu: "1"
        
        # Mounts
        volumeMounts:
        - name: model-cache
          mountPath: /root/.cache/huggingface
        - name: model-storage
          mountPath: /models
        - name: shm
          mountPath: /dev/shm
        
        # Health checks
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 300
          periodSeconds: 30
          timeoutSeconds: 10
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 120
          periodSeconds: 10
          timeoutSeconds: 5
        
        startupProbe:
          httpGet:
            path: /health
            port: 8000
          failureThreshold: 100
          periodSeconds: 10
      
      # Shared memory for GPU
      volumes:
      - name: shm
        emptyDir:
          medium: Memory
          sizeLimit: 16Gi
      
      - name: model-cache
        emptyDir:
          sizeLimit: 20Gi
      
      - name: model-storage
        persistentVolumeClaim:
          claimName: model-storage

---
# Service
apiVersion: v1
kind: Service
metadata:
  name: vllm-service
  namespace: llm-serving
  labels:
    app: vllm
spec:
  type: LoadBalancer
  selector:
    app: vllm
  ports:
  - name: http
    port: 8000
    targetPort: 8000
  - name: metrics
    port: 8001
    targetPort: 8001

---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: vllm-hpa
  namespace: llm-serving
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: vllm-server
  
  minReplicas: 2
  maxReplicas: 4
  
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
```

**Deploy:**
```bash
kubectl apply -f vllm-deployment.yaml

# Monitor rollout
kubectl rollout status deployment/vllm-server -n llm-serving

# Check status
kubectl get pods -n llm-serving
kubectl logs -f deployment/vllm-server -n llm-serving
```

---

## Model Serving with vLLM

### Testing the Deployment

```bash
# Port-forward for local testing
kubectl port-forward svc/vllm-service 8000:8000 -n llm-serving

# Test inference locally
curl http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/Llama-2-7b-chat-hf",
    "prompt": "What is machine learning?",
    "max_tokens": 256,
    "temperature": 0.7
  }' | jq
```

### API Endpoint Examples

```python
# Python client example
import requests
import json

VLLM_ENDPOINT = "http://vllm-service.llm-serving.svc.cluster.local:8000"

# Text completion
def get_completion(prompt, model="meta-llama/Llama-2-7b-chat-hf"):
    response = requests.post(
        f"{VLLM_ENDPOINT}/v1/completions",
        json={
            "model": model,
            "prompt": prompt,
            "max_tokens": 256,
            "temperature": 0.7,
            "top_p": 0.95
        }
    )
    return response.json()["choices"][0]["text"]

# Chat completion
def get_chat_response(messages, model="meta-llama/Llama-2-7b-chat-hf"):
    response = requests.post(
        f"{VLLM_ENDPOINT}/v1/chat/completions",
        json={
            "model": model,
            "messages": messages,
            "max_tokens": 256,
            "temperature": 0.7
        }
    )
    return response.json()["choices"][0]["message"]["content"]

# Example usage
result = get_chat_response([
    {"role": "user", "content": "Explain quantum computing"}
])
print(result)
```

---

## Monitoring and Management

### Prometheus Monitoring

```yaml
# File: prometheus-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: llm-serving
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    
    scrape_configs:
    - job_name: 'vllm'
      static_configs:
      - targets: ['vllm-service:8001']
    
    - job_name: 'kubernetes-nodes'
      kubernetes_sd_configs:
      - role: node
    
    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
```

### Grafana Dashboard

```json
{
  "dashboard": {
    "title": "LLM Inference Metrics",
    "panels": [
      {
        "title": "Request Latency (P95)",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, vllm_request_latency_ms)"
          }
        ]
      },
      {
        "title": "GPU Memory Usage %",
        "targets": [
          {
            "expr": "nvidia_smi_memory_used_mb / nvidia_smi_memory_total_mb * 100"
          }
        ]
      },
      {
        "title": "Tokens Generated Per Second",
        "targets": [
          {
            "expr": "rate(vllm_tokens_generated_total[1m])"
          }
        ]
      },
      {
        "title": "Queue Depth",
        "targets": [
          {
            "expr": "vllm_queue_size"
          }
        ]
      }
    ]
  }
}
```

### Aria-Native Monitoring

```bash
# Configure Aria to collect metrics
aria monitoring add-job \
  --name vllm-metrics \
  --collection-interval 30s \
  --target vllm-service.llm-serving.svc.cluster.local:8001

# Set up alerts
aria alert create \
  --name high-latency \
  --condition "vllm_request_latency_ms{quantile='0.95'} > 2000" \
  --action notify-ops-team
```

---

## Cost Analysis

### Hardware Cost Breakdown (3-year ownership)

```
Infrastructure Investment:
- 4 compute nodes @ $18k each:           $72,000
- 4 GPU cards (A100) @ $10k each:        $40,000
- Storage system (500TB):                $20,000
- Networking equipment:                  $15,000
- Power & cooling infrastructure:         $8,000
- Rack space & deployment:               $5,000
—————————————————————
Total Hardware:                          $160,000

Software Licenses:
- VMware vSphere Enterprise+:            $15,000
- VMware Aria Automation:                $8,000
- Support contracts (3 years):           $20,000
—————————————————————
Total Software:                          $43,000

Total Capital Expenditure:               $203,000
```

### Operational Costs (Monthly)

```
Personnel:
- Infrastructure Engineer (50%):         $2,500
- System Administrator (30%):            $1,500
- Backup/Monitoring (20%):               $1,000
—————————————————————
Staffing:                                $5,000

Infrastructure:
- Power consumption:                     $1,500
- Cooling:                                $800
- Network bandwidth:                      $300
- Maintenance & support:                 $400
—————————————————————
Infrastructure:                          $3,000

Software:
- License renewals:                      $1,200
- Monitoring tools (Prometheus, etc):     $200
—————————————————————
Software:                                $1,400

Total Monthly Operations:                $9,400
```

### Cost per Inference

```
At 1 million inferences/month:
Monthly operational cost:        $9,400
Cost per inference:              $0.0094

At 5 million inferences/month:
Monthly operational cost:        $10,500 (scale ~10%)
Cost per inference:              $0.0021

Comparison to cloud:
- AWS Bedrock:       $0.01-0.015 per inference
- Azure OpenAI:      $0.008-0.020 per inference
- On-premise:        $0.002-0.009 per inference

3-year Total Cost of Ownership:
On-premise:          $203,000 + ($9,400 × 36) = $541,400
Cloud (5M req/mo):   ($15 × 1000 × 36) = $540,000

Result: On-premise breaks even at ~4M requests/month
```

---

## Security & Compliance

### Data Sovereignty

```yaml
# File: network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: llm-serving-isolation
  namespace: llm-serving
spec:
  podSelector:
    matchLabels:
      app: vllm
  policyTypes:
  - Ingress
  - Egress
  
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: api-gateway
    ports:
    - protocol: TCP
      port: 8000
  
  egress:
  # Only allow DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
  
  # Allow to local databases only
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
```

### HIPAA Compliance Checklist

```
☑ Data Encryption
  - At rest: AES-256
  - In transit: TLS 1.3
  - Key management: HSM or secrets manager

☑ Access Controls
  - RBAC enabled in Kubernetes
  - Service account isolation
  - MFA for admin access

☑ Audit Logging
  - All API calls logged
  - Tampering detection
  - Retention: 6+ years

☑ Backup & Recovery
  - Daily automated backups
  - Tested restore procedures
  - Geographic separation

☑ Vulnerability Management
  - Regular vulnerability scans
  - Patch management process
  - Security incident response

☑ Business Associate Agreement (BAA)
  - Signed with all third-party vendors
  - Subcontractor requirements defined
```

---

## Disaster Recovery

### Backup Strategy

```bash
#!/bin/bash
# File: daily-backup.sh

BACKUP_DIR="/backups/llm-dr"
DATE=$(date +%Y%m%d)

# 1. Backup Kubernetes configuration
kubectl get all -A -o yaml > $BACKUP_DIR/k8s-config-$DATE.yaml

# 2. Backup persistent data
# Using VMware snapshot for PVCs
vmware-cli snapshot create \
  --vm llm-worker-1 \
  --name backup-$DATE

# 3. Backup database
pg_dump -h postgres.llm-serving \
  -U admin -d llm_db \
  | gzip > $BACKUP_DIR/db-backup-$DATE.sql.gz

# 4. Upload to secondary site
rsync -av $BACKUP_DIR \
  backup-user@secondary-site:/backups/llm

echo "Backup complete: $DATE"
```

### High Availability Setup

```
Primary Datacenter (Active):
- 2 control plane nodes
- 3 worker nodes with GPUs
- Primary database

Secondary Datacenter (Standby):
- 1 control plane node
- 1 worker node (small GPU)
- Replicated database

Failover Time: <5 minutes
Recovery Point Objective (RPO): <1 hour
Recovery Time Objective (RTO): <5 minutes
```

---

## Troubleshooting

### Common Issues

**Issue: Pods not getting GPU access**
```bash
# Check GPU operator status
kubectl get pods -n gpu-operator-system

# Check NVIDIA runtime
kubectl describe node | grep -A 5 "Allocatable"

# Fix: Reinstall GPU operator
helm uninstall gpu-operator
helm install gpu-operator nvidia/gpu-operator
```

**Issue: vLLM pod stuck in "Pending"**
```bash
# Check events
kubectl describe pod vllm-server-0 -n llm-serving

# Common causes:
# 1. Insufficient GPU: Scale down replicas
kubectl scale deployment vllm-server --replicas=1

# 2. Insufficient memory: Increase node memory
# 3. GPU in use: Check other pods
kubectl get pods --all-namespaces | grep gpu
```

**Issue: High latency (>5 seconds per inference)**
```bash
# Check GPU utilization
kubectl logs -n llm-serving vllm-server-0 | grep "GPU Utilization"

# Check queue depth
curl http://vllm-service:8000/stats | grep queue

# Solutions:
# 1. Scale up replicas
# 2. Use smaller model (7B vs 70B)
# 3. Enable prefix caching
# 4. Reduce max_tokens limit
```

**Issue: Out of memory (OOM) errors**
```bash
# Check GPU memory
kubectl exec -it vllm-server-0 -n llm-serving -- nvidia-smi

# Reduce batch size
# Modify vLLM deployment:
# - Reduce GPU_MEMORY_UTILIZATION from 0.9 to 0.7
# - Reduce max_model_len

# Or use quantization
# Convert model to 8-bit or 4-bit
# Reduces memory by 50-75%
```

---

## Production Checklist

- [ ] Hardware validated and stress-tested
- [ ] Network latency <1ms between GPU and CPU
- [ ] vSphere HA configured
- [ ] Kubernetes HA (3+ control planes)
- [ ] Backup solution tested and scheduled
- [ ] Monitoring and alerting configured
- [ ] Security policies applied
- [ ] Performance baseline established
- [ ] Load testing completed (>1M req/hour)
- [ ] Disaster recovery plan tested
- [ ] Staff trained on operations
- [ ] Documentation complete
- [ ] Compliance audit completed
- [ ] Insurance/liability reviewed

---

## Next Steps

1. **Assess infrastructure:** Do you have existing VMware infrastructure?
2. **Capacity planning:** Estimate requests/month and model size
3. **Pilot deployment:** Start with 1-2 GPUs to test
4. **Scale gradually:** Add capacity based on demand
5. **Optimize costs:** Monitor and adjust resource allocation

---

**For regulated industries, on-premise deployment provides:** ✓ Data sovereignty ✓ Compliance ✓ Cost efficiency at scale

