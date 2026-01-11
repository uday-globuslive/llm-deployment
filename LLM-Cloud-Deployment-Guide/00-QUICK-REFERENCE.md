# Quick Reference Guide

## Cloud Platform Comparison at a Glance

### For Different Budgets

**Startup / PoC ($500-2000/month)**
- AWS Bedrock (low volume)
- GCP Vertex AI (free tier Gemini Pro)
- Azure OpenAI with GPT-3.5 Turbo

**Growing Startup ($2000-10000/month)**
- AWS Bedrock + optimization
- Azure Container Instances
- SageMaker with smaller models

**Scale ($10000+/month)**
- Self-hosted EC2 / GKE / AKS
- Kubernetes orchestration
- Multi-region setup

---

## Quick Checklist: Pre-Deployment

- [ ] **Architecture**: Chose deployment pattern (managed vs self-hosted)?
- [ ] **Scaling**: Auto-scaling configured?
- [ ] **Caching**: Response caching enabled?
- [ ] **Monitoring**: Dashboards set up?
- [ ] **Alerts**: Alarms configured (latency, errors, cost)?
- [ ] **Security**: Authentication enabled?
- [ ] **Rate Limiting**: Rate limits set?
- [ ] **Backup**: Backup/disaster recovery plan?
- [ ] **Compliance**: Required regulations met?
- [ ] **Testing**: Load tested at 2x expected traffic?

---

## Common Issues and Solutions

### Issue 1: "Slow Responses (>10 seconds)"

**Diagnosis:**
```bash
# Check where time is spent
# Is it LLM or infrastructure?

# Check LLM latency
time curl -X POST https://your-api/chat \
  -H "Content-Type: application/json" \
  -d '{"prompt":"hi"}'
```

**Solutions (in order of impact):**

1. **Upgrade model (if self-hosted)**
   - Switching GPU can save 50% latency
   - T4 → A10 → A100

2. **Use streaming responses**
   - Send tokens as generated vs waiting for complete response

3. **Reduce max_tokens**
   - Generate less text = faster

4. **Implement caching**
   - Same questions answered from cache

5. **Use smaller model**
   - Trade-off: Quality vs speed
   - 7B vs 70B = 10x faster

---

### Issue 2: "High Error Rate (> 1%)"

**Check error types:**
```bash
# CloudWatch Logs
aws logs filter-log-events \
  --log-group-name /aws/lambda/llm-app \
  --filter-pattern "ERROR"

# Look for patterns:
# 1. ThrottlingException → Rate limit exceeded
# 2. ValidationException → Invalid input
# 3. InternalServerError → LLM service issue
# 4. Timeout → Response took too long
```

**Solutions:**

| Error | Cause | Fix |
|-------|-------|-----|
| ThrottlingException | Too many requests | Reduce load, use rate limiting |
| ValidationException | Bad input | Add input validation |
| InternalServerError | LLM service down | Implement fallback model |
| Timeout | Slow response | Increase timeout, upgrade GPU |
| OutOfMemory | Model too large | Use smaller model, increase memory |

---

### Issue 3: "Cost Overrun"

**First: Understand where money goes**
```bash
# AWS Cost Explorer
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity MONTHLY \
  --filter '{...}' \
  --metrics "UnblendedCost"
```

**Quick cost reduction (achievable in days):**
1. Implement caching (40-60% reduction)
2. Reduce response length (20-30% reduction)
3. Batch similar requests (20% reduction)

**Medium-term (weeks):**
1. Switch to cheaper model (50% reduction)
2. Auto-scaling and scheduled scaling (20-40% reduction)
3. Use reserved capacity (20-30% discount)

**Long-term (months):**
1. Fine-tune smaller model (70-90% reduction)
2. Self-host instead of managed service (50-80% reduction)

---

### Issue 4: "Model Hallucinating / Bad Quality"

**Root causes:**
```
- Prompt not specific enough
- Wrong model for task
- Temperature too high
- Not enough context provided
```

**Solutions:**

```python
# 1. Improve prompt specificity
BAD: "Write a poem"
GOOD: "Write a short funny poem about DevOps in 50 words, rhyme scheme AABB"

# 2. Lower temperature
response = openai.ChatCompletion.create(
    ...
    temperature=0.1  # Lower = more consistent
)

# 3. Add context
SYSTEM = """You are a technical expert.
Only answer based on provided documentation.
If unsure, say 'I don't know'."""

# 4. Use better model
"gpt-4" vs "gpt-3.5-turbo"
"claude-3-sonnet" vs "claude-3-haiku"
```

---

## Performance Benchmarks

### Model Speed Comparison (Approximate)

| Model | Time (ms) | Quality | Notes |
|-------|-----------|---------|-------|
| Llama 2 7B | 500 | ⭐⭐⭐ | Fast, good for real-time |
| Llama 2 13B | 800 | ⭐⭐⭐⭐ | Good balance |
| Llama 2 70B | 2000 | ⭐⭐⭐⭐⭐ | Slow, excellent quality |
| GPT-3.5 | 1200 | ⭐⭐⭐⭐ | Reliable, moderate speed |
| GPT-4 | 2000 | ⭐⭐⭐⭐⭐ | Slowest, best quality |
| Claude 3 Haiku | 800 | ⭐⭐⭐⭐ | Fastest, good quality |

### Throughput Comparison (Tokens/second)

| Setup | Throughput |
|-------|-----------|
| CPU-only | 5-10 tokens/sec |
| Single GPU (T4) | 50-100 tokens/sec |
| Single GPU (A10) | 100-200 tokens/sec |
| Single GPU (A100) | 200-400 tokens/sec |
| Managed Service | Variable, hidden from you |

---

## Cost Estimation Calculator

### Simple Formula

```
Monthly Cost = (Daily Requests × 30) × Cost Per Request

Cost Per Request = 
  (Input Tokens × Input Rate) + 
  (Output Tokens × Output Rate)

Example:
- 10,000 daily requests
- 150 input tokens, 200 output tokens per request
- Claude 3 Sonnet: $0.003/1K input, $0.015/1K output

Cost Per Request = (150 × 0.003/1000) + (200 × 0.015/1000)
                 = $0.00045 + $0.003
                 = $0.00345

Monthly Cost = 10,000 × 30 × $0.00345 = $1,035/month
```

### Infrastructure Costs

```
Self-Hosted EC2 + vLLM:
- Instance cost: $0.55/hour × 730 hours = $401.50/month
- Storage (100GB): $10/month
- Transfer: $0-50/month
- Total: ~$500/month

Auto-scaling (3 instances, 50% average):
- 1.5 instances × $401.50 = $602/month

Kubernetes (3 nodes):
- 3 × $1.00/hour × 730 = $2,190/month
- Plus persistent storage, networking
- Total: ~$2,500/month+
```

---

## Platform-Specific Quick Commands

### AWS

```bash
# List Bedrock models
aws bedrock list-foundation-models --region us-east-1

# Create SageMaker endpoint
aws sagemaker create-endpoint --endpoint-name llm --endpoint-config-name llm-config

# Check Lambda logs
aws logs tail /aws/lambda/my-function --follow

# Update cost budget
aws budgets update-budget --budget-name llm-budget --new-budget-limit 5000
```

### Azure

```bash
# List Azure OpenAI models
az cognitiveservices account deployment list \
  --resource-group mygroup \
  --name myresource

# Deploy model
az cognitiveservices account deployment create \
  --resource-group mygroup \
  --name myresource \
  --deployment-name gpt35-deployment \
  --model-name gpt-35-turbo

# Check costs
az billing invoice list --max-items 3

# Create AKS cluster
az aks create --resource-group mygroup --name my-cluster
```

### GCP

```bash
# List Vertex AI models
gcloud ai models list

# Deploy to Cloud Run
gcloud run deploy llm-service --source .

# Check costs
gcloud billing budgets create --billing-account ACCOUNT --budget-amount 5000

# Create GKE cluster
gcloud container clusters create llm-cluster --zone us-central1-a
```

---

## Monitoring Queries

### CloudWatch (AWS)

```python
# Get error rate for last hour
cloudwatch.get_metric_statistics(
    Namespace='LLM-App',
    MetricName='ErrorRate',
    StartTime=datetime.now() - timedelta(hours=1),
    EndTime=datetime.now(),
    Period=300,
    Statistics=['Average']
)

# Get token usage cost
cloudwatch.get_metric_statistics(
    Namespace='LLM-App',
    MetricName='TokenCost',
    StartTime=datetime.now() - timedelta(days=1),
    EndTime=datetime.now(),
    Period=3600,
    Statistics=['Sum']
)
```

### Application Insights (Azure)

```kusto
// Get P95 latency
requests
| where timestamp > ago(24h)
| extend duration_ms = duration
| summarize p95_latency = percentile(duration_ms, 95) by bin(timestamp, 1h)

// Get error count by type
requests
| where timestamp > ago(24h) and success == false
| summarize count() by resultCode
```

### Cloud Logging (GCP)

```
// High error rate
resource.type="cloud_function"
severity="ERROR"
timestamp>="2024-01-11T00:00:00Z"

// High latency
resource.type="cloud_function"
duration>5000
```

---

## DevOps Decision Matrix

**When to choose managed service:**
- ✅ Quick time-to-market needed
- ✅ Don't have DevOps team
- ✅ Unpredictable traffic
- ✅ Multiple models needed
- ✅ Don't want ops overhead

**When to choose self-hosted:**
- ✅ High volume (cost matters)
- ✅ Custom inference framework
- ✅ Need full control
- ✅ Experienced DevOps team
- ✅ Long-term commitment

**When to choose hybrid:**
- ✅ Varies by use case
- ✅ Important use cases: self-hosted
- ✅ Secondary use cases: managed
- ✅ Testing: managed service
- ✅ Production: self-hosted

---

## Essential Links

### AWS
- [Bedrock Documentation](https://docs.aws.amazon.com/bedrock/)
- [SageMaker Documentation](https://docs.aws.amazon.com/sagemaker/)
- [CloudWatch Documentation](https://docs.aws.amazon.com/cloudwatch/)

### Azure
- [Azure OpenAI Documentation](https://learn.microsoft.com/en-us/azure/ai-services/openai/)
- [AKS Documentation](https://learn.microsoft.com/en-us/azure/aks/)
- [Azure Monitor Documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/)

### GCP
- [Vertex AI Documentation](https://cloud.google.com/vertex-ai/docs)
- [Cloud Run Documentation](https://cloud.google.com/run/docs)
- [GKE Documentation](https://cloud.google.com/kubernetes-engine/docs)

---

## Common Acronyms Explained

| Acronym | Meaning | Context |
|---------|---------|---------|
| **LLM** | Large Language Model | AI model for text generation |
| **RTO** | Recovery Time Objective | How long service can be down |
| **RPO** | Recovery Point Objective | How much data can be lost |
| **QPS** | Queries Per Second | Request throughput |
| **P50, P95, P99** | Percentile | Latency metrics |
| **SLA** | Service Level Agreement | Uptime guarantee |
| **TTL** | Time To Live | Cache expiration time |
| **PII** | Personally Identifiable Info | Sensitive data |
| **TPS** | Tokens Per Second | Generation speed |
| **VRAM** | Video RAM | GPU memory |
| **Quantization** | Model compression | Reduce model size |
| **Fine-tuning** | Model customization | Train on specific data |

---

## Next Steps Recommendation

**Week 1:**
- [ ] Choose deployment platform (AWS/Azure/GCP)
- [ ] Set up basic deployment
- [ ] Get it working with small test

**Week 2:**
- [ ] Set up monitoring and alerts
- [ ] Test with expected load
- [ ] Document runbooks

**Week 3:**
- [ ] Optimize costs
- [ ] Add security (auth, rate limit)
- [ ] Set up backup/disaster recovery

**Week 4+:**
- [ ] Monitor production metrics
- [ ] Optimize based on real usage
- [ ] Plan capacity for growth

---

**Remember:** Start simple, add complexity as needed. Don't over-engineer before you have real requirements.
