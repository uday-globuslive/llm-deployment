# Cost Optimization Strategies for LLM Deployments

## Cost Breakdown

**Typical LLM deployment costs:**

```
Monthly Breakdown:
├─ Compute (GPU instances): 60-70% of cost
├─ Model API calls: 20-30% of cost
├─ Storage (models, logs): 3-5% of cost
├─ Network bandwidth: 1-2% of cost
└─ Monitoring/Logging: 2-3% of cost
```

**Real example: $10,000/month deployment**
```
GPU instances:    $6,500 (65%)
Model API calls:  $2,500 (25%)
Storage:          $400 (4%)
Network:          $200 (2%)
Monitoring:       $400 (4%)
Total:            $10,000
```

---

## Cost Drivers and How to Optimize

### 1. Model Selection (30% savings possible)

**Problem:** Larger models cost more

**Trade-off Table:**

| Model | Size | Speed | Quality | Cost/1M Tokens |
|-------|------|-------|---------|---|
| Llama 2 7B | Small | ✅✅✅ | ✅✅ | $0.10 |
| Llama 2 13B | Medium | ✅✅ | ✅✅✅ | $0.20 |
| Llama 2 70B | Large | ✅ | ✅✅✅✅ | $1.00 |
| GPT-3.5 Turbo | Variable | ✅✅ | ✅✅✅ | $0.65 |
| GPT-4 | Variable | ✅ | ✅✅✅✅ | $30.00 |

**Optimization strategies:**

**Strategy A: Use Smaller Models**
```
Current: GPT-4 for all requests ($30/1M tokens)
Better: Use GPT-3.5 for 70% of requests ($0.65/1M)
        Use GPT-4 for 30% of requests ($30/1M)

Cost reduction: 75%
Quality impact: Minimal
```

**Implementation:**
```python
def choose_model(task_complexity):
    if task_complexity == "simple":  # Q&A, summarization
        return "gpt-35-turbo"  # Cheaper
    else:  # Complex reasoning, code gen
        return "gpt-4"  # Better quality
```

**Strategy B: Use Quantized Models (Self-Hosted)**
```
Original:  70B parameter model
Quantized: Same model, 4-bit quantization (4x smaller)
           Runs on 1 GPU instead of 4
           Speed: 30% slower
           Cost: 75% reduction

Good for: Non-real-time workloads
```

**Strategy C: Fine-Tuning (Long-term)**
```
Start: Using general-purpose model
Goal: Build specific-purpose model

Train: Small model on your task
       Takes time but worth it
Cost: One-time training cost (~$100-1000)
Benefit: 90% smaller model, 99% accuracy

Break-even: After 2-3 months
```

---

### 2. Request Optimization (20% savings possible)

**Problem:** Large prompts cost more (token count)

**Cost calculation:**
```
Prompt tokens: "What is DevOps?" = 6 tokens = $0.00002
Response: 256 tokens = $0.0048
Total: $0.00482 per request

Now × 1M requests/month = $4,820/month
```

**Optimization strategies:**

**Strategy A: Prompt Optimization**
```python
# Bad: Verbose prompt
prompt = """
You are a helpful assistant that answers questions
about DevOps. DevOps is a set of practices...
[long context]
User question: What is DevOps?
"""
# Tokens: 150

# Good: Concise prompt
prompt = "Q: What is DevOps?\nA:"
# Tokens: 8

# Savings: 94% reduction in prompt tokens
```

**Strategy B: Use System Prompt Only Once**
```python
# Instead of including context in each request
# Bad:
for question in questions:
    response = llm(context + "\n" + question)

# Good:
system_message = context
for question in questions:
    response = llm(
        system=system_message,
        user=question
    )
# Saves repeating context for each request
```

**Strategy C: Caching**
```python
# Without caching
100 users ask "What is DevOps?"
Cost: 100 requests × $0.005 = $0.50

# With 1-hour caching
First request: $0.005
Next 99 requests: Cache hit = $0.00
Total: $0.005 (99% savings)

Implementation:
redis_key = hash(prompt)
if cache.exists(redis_key):
    return cache.get(redis_key)
result = llm(prompt)
cache.set(redis_key, result, timeout=3600)
```

**Strategy D: Reduce Response Length**
```python
# Instead of generating 2000-token response
# Ask for shorter response:

response = openai.ChatCompletion.create(
    model="gpt-35-turbo",
    messages=[{"role": "user", "content": prompt}],
    max_tokens=256  # Limit response length
)

# Savings: 87% reduction (2000 → 256 tokens)
```

**Strategy E: Batch Processing**
```python
# Instead of API calls:
# 1000 requests × 1 second = 1000 seconds = 17 minutes

# Use batch API:
# 1000 requests processed together = 1 minute

# Cost: 50% lower per token (batch rates)
# Speed: 20x faster
# Trade-off: Not real-time (results in 1 hour)

# Good for: Report generation, data processing
```

**Implementation:**
```python
import json

# Prepare batch requests
batch_requests = []
for text in documents:
    batch_requests.append({
        "custom_id": f"request-{i}",
        "params": {
            "model": "gpt-35-turbo",
            "messages": [{"role": "user", "content": f"Summarize: {text}"}]
        }
    })

# Save to file
with open("batch_requests.jsonl", "w") as f:
    for req in batch_requests:
        f.write(json.dumps(req) + "\n")

# Submit batch
batch = client.batches.create(
    input_file_id="file-123",
)

# Check results later
batch_status = client.batches.retrieve(batch.id)
# Returns all results when completed
```

---

### 3. Infrastructure Optimization (40% savings possible)

**Problem:** Paying for unused compute

**Option 1: Use Managed Services Wisely**

```python
# Bad: Bedrock for high volume
# 1M tokens/month: $4,000/month

# Better: EC2 + vLLM
# Same workload: $500-1000/month
# Break-even: 1-2 months

# When Bedrock makes sense:
# - Proof of concept (no long-term commitment)
# - Unpredictable traffic (no upfront cost)
# - Multiple models needed
# - Don't want ops overhead
```

**Option 2: Reserved Capacity (AWS/GCP)**

```
AWS Bedrock Provisioned Throughput:
- On-demand: $0.003 per input token
- Reserved: $3.5/hour (1000 tpm guarantee)
  = $2,520/month

Break-even: After 840M input tokens/month
(Or ~$2,500 in costs)

If you're spending > $2,500/month on Bedrock → Get reserved
```

**Option 3: Use Right-Sized Instances**

```
Bad:
- EC2 g4dn.12xlarge (4 GPUs): $6.48/hour
- Only using 20% capacity
- Cost: $6.48/hour for 20% usage

Better:
- EC2 g4dn.xlarge (1 GPU): $0.55/hour
- Running at 80% capacity
- Cost: $0.55/hour for 80% usage

Savings: 88%
```

**Option 4: Auto-Scaling**

```python
# Bad: Always running 10 instances
# Cost: $10 × $1/hour = $10/hour (24/7)
#       = $7,200/month

# Better: Scale based on traffic
# Off-peak: 2 instances
# Peak: 10 instances
# Average: 5 instances
# Cost: $5 × $1/hour × 730 hours = $3,650/month

# Savings: 49%

# Implementation:
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name llm-asg \
  --policy-name scale-up \
  --scaling-adjustment 2 \
  --adjustment-type ChangeInCapacity

# Trigger when CPU > 70%
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name llm-asg \
  --policy-name scale-down \
  --scaling-adjustment -1 \
  --adjustment-type ChangeInCapacity

# Trigger when CPU < 30%
```

**Option 5: Scheduled Scaling**

```bash
# Scale down for non-business hours
# 6 PM - 8 AM: Run 2 instances
# 8 AM - 6 PM: Run 10 instances

# AWS CLI
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name llm-asg \
  --scheduled-action-name scale-down-night \
  --recurrence "0 18 * * *" \
  --min-size 2 \
  --max-size 2

aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name llm-asg \
  --scheduled-action-name scale-up-day \
  --recurrence "0 8 * * *" \
  --min-size 10 \
  --max-size 10

# Savings: 60% if 60% off-peak traffic
```

**Option 6: Use Spot Instances**

```
Regular EC2: $0.55/hour
Spot EC2: $0.17/hour (69% discount)

Cost for 100 hours:
Regular: $55
Spot: $17

Risk: Can be interrupted (reclaimed by AWS)
Solution: Use for non-critical workloads

Good for: Batch processing, testing, non-essential services
Bad for: Production customer-facing chat
```

---

### 4. Storage Optimization (10% savings possible)

**Problem:** Model files and logs take space

**Optimization strategies:**

**Strategy A: Use Quantized Model Weights**
```
Llama 2 70B full precision: 140 GB
Llama 2 70B 4-bit quantized: 35 GB
Savings: 75%

Storage cost: $23.33/month → $5.83/month
Inference speed: 30% slower (acceptable for most)
```

**Strategy B: Delete Old Logs**
```python
# Logs grow quickly with high-volume services
# Typical: 1 GB per 100K requests

# Solution: Archive old logs
# Keep: Last 7 days in fast storage
# Archive: > 7 days in cheaper storage

import boto3
from datetime import datetime, timedelta

s3 = boto3.client('s3')

# Transition to Glacier (99% cheaper after 30 days)
s3.put_bucket_lifecycle_configuration(
    Bucket='llm-logs',
    LifecycleConfiguration={
        'Rules': [{
            'Id': 'archive-old-logs',
            'Status': 'Enabled',
            'Transitions': [{
                'Days': 30,
                'StorageClass': 'GLACIER'
            }],
            'Expiration': {
                'Days': 90  # Delete after 90 days
            }
        }]
    }
)

# Savings: 95% on log storage > 30 days old
```

**Strategy C: Share Model Cache**

```
Bad:
- Server 1: Loads Llama 2 70B = 140 GB memory
- Server 2: Loads Llama 2 70B = 140 GB memory
- Total: 280 GB across servers

Better:
- Shared network storage: 140 GB
- Both servers access same copy
- Savings: 140 GB

Implementation:
- Use NFS (Network File System)
- Or: Use shared object storage (S3/Blob)
```

---

## Cost Monitoring and Budgeting

### Set Budget Alerts

**AWS:**
```bash
# Create budget alert
aws budgets create-budget \
  --account-id 123456789 \
  --budget '{
    "BudgetName": "LLM-Monthly",
    "BudgetLimit": {
      "Amount": "5000",
      "Unit": "USD"
    },
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST"
  }' \
  --notifications-with-subscribers '{
    "member": {
      "Notification": {
        "NotificationType": "FORECASTED",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 80
      },
      "Subscribers": [{
        "SubscriptionType": "EMAIL",
        "Address": "finance@company.com"
      }]
    }
  }'
```

**Azure:**
```
Subscriptions → Cost Management → Budgets
→ Create Budget
→ Set amount: $5,000
→ Alert at: 80%, 100%
→ Notify: DevOps team
```

**GCP:**
```bash
# Create budget
gcloud billing budgets create \
  --billing-account=ACCOUNT_ID \
  --display-name="LLM Budget" \
  --budget-amount=5000 \
  --threshold-rule=percent=80 \
  --threshold-rule=percent=100
```

### Track Costs by Service

```python
# Log service and cost
def log_cost_metrics(service, tokens_used, cost):
    metrics = {
        "timestamp": datetime.now(),
        "service": service,  # "bedrock", "openai", etc
        "tokens": tokens_used,
        "cost": cost,
        "model": model_name,
        "user_id": user_id
    }
    
    # Store in database for analysis
    cost_db.insert(metrics)
    
    # Track by service
    cloudwatch.put_metric_data(
        Namespace='LLM-Costs',
        MetricData=[
            {
                'MetricName': f'{service}_daily_cost',
                'Value': cost,
                'Unit': 'None',
                'Dimensions': [
                    {'Name': 'Service', 'Value': service}
                ]
            }
        ]
    )

# Example calls
log_cost_metrics('bedrock', 150, 0.00475)
log_cost_metrics('openai', 200, 0.00585)
```

### Monthly Cost Review Template

```markdown
## Monthly Cost Review - January 2024

### Summary
- Budget: $5,000
- Actual: $4,234
- Variance: -15.3% (GOOD)

### Breakdown
| Service | Cost | % of Total | Trend |
|---------|------|-----------|-------|
| Compute | $2,500 | 59% | ↓ 10% |
| API Calls | $1,000 | 24% | → Same |
| Storage | $400 | 9% | ↑ 5% |
| Network | $200 | 5% | → Same |
| Monitoring | $134 | 3% | ↓ 15% |

### Actions for Next Month
- [ ] Reduce compute: Switch 20% of traffic to spot instances
- [ ] Reduce API costs: Implement caching
- [ ] Review storage: Archive old logs
- [ ] Target savings: $500 (10% reduction)
```

---

## Cost Optimization Roadmap

### Month 1: Quick Wins (5-20% savings)
- [ ] Implement caching
- [ ] Reduce response length limits
- [ ] Set budget alerts
- [ ] Schedule scaling for off-peak hours
- **Expected savings:** $500-2,000/month

### Month 2: Model Optimization (20-40% savings)
- [ ] Switch to smaller models where possible
- [ ] Implement smart model selection
- [ ] Use batch API for non-real-time work
- [ ] Consider quantized models
- **Expected savings:** $1,000-4,000/month

### Month 3: Infrastructure Optimization (20-50% savings)
- [ ] Right-size instances
- [ ] Enable auto-scaling
- [ ] Consider reserved capacity
- [ ] Use spot instances for appropriate workloads
- **Expected savings:** $1,000-5,000/month

### Month 6+: Advanced Optimization (50%+ savings)
- [ ] Fine-tune models for your task
- [ ] Multi-model architecture
- [ ] Geographic optimization
- [ ] Custom inference framework
- **Expected savings:** $2,000-10,000+/month

---

## Real-World Cost Optimization Example

**Company: TechCorp**
**Initial Setup:** ChatBot for 10K daily users

**Month 1 Costs:**
```
Bedrock API calls:  $5,000
EC2 compute:        $2,000
Storage:            $300
Total:              $7,300/month
```

**Optimizations Applied:**
1. Switch from Claude 3 Sonnet to Claude 3 Haiku (50% cheaper)
2. Implement caching (reduces API calls by 40%)
3. Batch process non-real-time requests (30% of traffic)
4. Archive old logs (reduce storage)

**Month 3 Costs:**
```
Bedrock API calls:  $1,750 (Claude 3 Haiku 70% cheaper, 40% fewer calls)
Batch processing:   $800 (cheaper than real-time)
EC2 compute:        $2,000 (same, stable load)
Storage:            $50 (archived old logs)
Total:              $4,600/month

Savings: $2,700/month (37%)
ROI on optimization: Immediate
```

---

## Cost Optimization Checklist

- [ ] Monitor daily costs
- [ ] Set budget alerts (80%, 100%)
- [ ] Implement caching (easy, 40-60% savings)
- [ ] Optimize prompts (medium, 20-30% savings)
- [ ] Right-size instances (medium, 20-40% savings)
- [ ] Enable auto-scaling (medium, 30-50% savings)
- [ ] Consider reserved capacity (medium, 20-30% savings)
- [ ] Use batch API for non-real-time (easy, 30% savings)
- [ ] Switch to smaller models where possible (hard, 50-80% savings)
- [ ] Review quarterly and adjust strategy

---

**Key Takeaway:** Start with quick wins (caching, prompts), then move to infrastructure changes (scaling, auto-scaling). Real savings come from architectural choices and right-sizing.

**Next:** See `07-Best-Practices/` for reliability and security considerations.
