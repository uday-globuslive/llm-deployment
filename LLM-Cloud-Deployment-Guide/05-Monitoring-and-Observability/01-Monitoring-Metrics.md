# Monitoring LLM Deployments

## Why Monitoring Matters

**Real-world incident:**
```
Tuesday 2PM: Chatbot starts responding with gibberish
Customer service calls spike
No one notices until phone lines overwhelmed
Cost: $50,000+ lost revenue, reputation damage

What went wrong: Model inference crashed, but no alert was set
```

**With proper monitoring:**
- Alert fires within 30 seconds
- On-call engineer gets paged
- Service restored in 5 minutes
- Cost: Near zero damage

---

## Key Metrics to Monitor

### 1. Availability Metrics

| Metric | What It Means | Target | Alert If |
|--------|---------------|--------|----------|
| **Uptime %** | Service availability | 99.9% | < 99% |
| **Error Rate** | % of failed requests | < 0.5% | > 1% |
| **API Response Status** | HTTP 200/5xx | All 200s | Any 5xx |
| **Model Service Up** | Is LLM responding? | 100% | Down |

**Why it matters:** Users can't access service = lost revenue

**How to implement:**

**AWS (CloudWatch):**
```python
# In your Lambda/application
import boto3

cloudwatch = boto3.client('cloudwatch')

try:
    # Your Bedrock call
    response = bedrock.invoke_model(...)
    # Success
    cloudwatch.put_metric_data(
        Namespace='LLM-App',
        MetricData=[
            {
                'MetricName': 'SuccessfulRequests',
                'Value': 1,
                'Unit': 'Count'
            }
        ]
    )
except Exception as e:
    # Error
    cloudwatch.put_metric_data(
        Namespace='LLM-App',
        MetricData=[
            {
                'MetricName': 'FailedRequests',
                'Value': 1,
                'Unit': 'Count'
            }
        ]
    )
```

**Azure (Application Insights):**
```python
from applicationinsights import TelemetryClient

client = TelemetryClient('instrumentation-key')

try:
    response = openai_client.chat.completions.create(...)
    client.track_event('ChatSuccess')
except Exception as e:
    client.track_exception()
```

**GCP (Cloud Logging):**
```python
from google.cloud import logging

logging_client = logging.Client()
logger = logging_client.logger('llm-app')

try:
    response = model.generate_content(prompt)
    logger.log_struct({
        "status": "success",
        "latency_ms": latency
    })
except Exception as e:
    logger.log_struct({
        "status": "error",
        "error": str(e)
    })
```

---

### 2. Performance Metrics

| Metric | What It Means | Target | Alert If |
|--------|---------------|--------|----------|
| **Latency (P50)** | 50% of requests finish within X ms | < 2 seconds | > 5 sec |
| **Latency (P95)** | 95% of requests finish within X ms | < 5 seconds | > 10 sec |
| **Latency (P99)** | 99% of requests finish within X ms | < 10 seconds | > 30 sec |
| **Throughput** | Requests per second | Planned level | Drop 20% |
| **Token Generation Speed** | Tokens/second | 50+ tokens/sec | < 10 |

**Why it matters:** Slow service = bad user experience = users leave

**Breaking down latencies:**
```
Total Latency (2.5 seconds) =
  - Network latency (0.1 sec): Time to reach API
  - Application processing (0.2 sec): Validation, logging
  - Model inference (1.8 sec): Actual LLM computation
  - Network return (0.4 sec): Response transmission
```

**How to measure:**

**AWS:**
```python
import time
import boto3

cloudwatch = boto3.client('cloudwatch')

start_time = time.time()

try:
    response = bedrock.invoke_model(...)
    
    latency = (time.time() - start_time) * 1000  # Convert to ms
    
    cloudwatch.put_metric_data(
        Namespace='LLM-App',
        MetricData=[
            {
                'MetricName': 'RequestLatency',
                'Value': latency,
                'Unit': 'Milliseconds'
            }
        ]
    )
except Exception as e:
    pass
```

---

### 3. Cost Metrics

| Metric | What It Means | Calculation | Alert If |
|--------|---------------|-------------|----------|
| **Tokens Used** | Total tokens (input + output) | Count from response | Spike |
| **Cost per Request** | How much each request costs | Tokens × Rate | > Budget |
| **Daily Cost** | Total spent today | Sum of all requests | > $500 |
| **Monthly Projection** | Cost by end of month | Daily × 30 | > Budget |

**Why it matters:** Unexpected costs = budget overrun = angry CFO

**How to track:**

**AWS Bedrock:**
```python
response = bedrock.invoke_model(...)
result = json.loads(response['body'].read())

input_tokens = result['usage']['input_tokens']
output_tokens = result['usage']['output_tokens']

# Claude pricing (as of Jan 2026)
cost = (input_tokens * 0.003 + output_tokens * 0.015) / 1000

# Log it
logger.info(f"Request cost: ${cost:.4f}, Tokens: {input_tokens + output_tokens}")
```

**Azure OpenAI:**
```python
response = client.chat.completions.create(...)

# GPT-3.5 Turbo pricing
input_tokens = response.usage.prompt_tokens
output_tokens = response.usage.completion_tokens

cost = (input_tokens * 0.0005 + output_tokens * 0.0015) / 1000
```

**GCP Vertex AI:**
```python
response = model.generate_content(prompt)

# PaLM pricing (approximate)
input_tokens = response.usage_metadata.prompt_token_count
output_tokens = response.usage_metadata.candidates_token_count

cost = (input_tokens * 0.0001 + output_tokens * 0.0003) / 1000
```

---

### 4. GPU/Resource Metrics

| Metric | What It Means | Target | Alert If |
|--------|---------------|--------|----------|
| **GPU Utilization %** | How much of GPU is used | 60-80% | < 30% or > 95% |
| **GPU Memory %** | GPU VRAM usage | 70-85% | > 90% |
| **CPU Utilization %** | CPU usage | 40-60% | > 85% |
| **Memory Utilization %** | RAM usage | 60-75% | > 90% |
| **Queue Length** | Waiting requests | < 10 | > 50 |

**Why it matters:**
- Low utilization = wasting money
- High utilization = performance issues
- Queue growing = need to scale

**How to measure:**

**AWS (EC2/Self-Hosted):**
```bash
# SSH into instance
nvidia-smi  # Check GPU utilization

# Get metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name GPUUtilization \
  --dimensions Name=InstanceId,Value=i-1234567 \
  --start-time 2024-01-11T00:00:00Z \
  --end-time 2024-01-11T01:00:00Z \
  --period 300 \
  --statistics Average
```

**Azure (VMs):**
```bash
# In VM
nvidia-smi  # GPU metrics
top         # CPU/Memory metrics

# In Azure Portal
Monitor → Metrics → Virtual Machine → GPUUtilization
```

**GCP (GKE):**
```bash
# Check pod GPU usage
kubectl top pod -A

# View GPU allocation
kubectl describe nodes
```

---

### 5. Business Metrics

| Metric | What It Means | Example Target |
|--------|---------------|-----------------|
| **Requests/Day** | How much the model is used | 10,000 |
| **Active Users** | Concurrent users | 100 |
| **Average Session Duration** | How long users interact | 10 minutes |
| **Task Success Rate** | % of tasks completed | 95% |
| **User Satisfaction** | Ratings/feedback | > 4.0/5.0 |

**Why it matters:** Tells you if the deployment is actually useful

**How to track:**

```python
# Log when user starts session
logger.info(f"Session start: user_id={user_id}")

# Log each interaction
logger.info(f"Interaction: user={user_id}, prompt_len={len(prompt)}, tokens_used={tokens}")

# Log when session ends
logger.info(f"Session end: user={user_id}, duration={duration_minutes}")

# Track satisfaction (if you have feedback mechanism)
satisfaction = request.json.get('rating')  # 1-5 stars
logger.info(f"Satisfaction: user={user_id}, rating={satisfaction}")
```

---

## Setting Up Dashboards

### AWS CloudWatch Dashboard

```python
import boto3

cloudwatch = boto3.client('cloudwatch')

# Create dashboard
dashboard_body = {
    'widgets': [
        {
            'type': 'metric',
            'properties': {
                'metrics': [
                    ['LLM-App', 'RequestLatency', {'stat': 'Average'}],
                    ['.', '.', {'stat': 'p99'}],
                    ['LLM-App', 'ErrorRate'],
                    ['LLM-App', 'TokensUsed'],
                    ['AWS/Bedrock', 'UserErrors']
                ],
                'period': 300,
                'stat': 'Average',
                'region': 'us-east-1',
                'title': 'LLM Performance'
            }
        }
    ]
}

cloudwatch.put_dashboard(
    DashboardName='LLM-Monitoring',
    DashboardBody=str(dashboard_body)
)
```

### Azure Application Insights Dashboard

```python
# In Azure Portal: Application Insights → Workbooks
# Create workbook with queries:

# Request rate
requests
| summarize count() by bin(timestamp, 1m)

# Average latency
requests
| extend latency = duration
| summarize avg(latency) by bin(timestamp, 1m)

# Error rate
requests
| where success == false
| summarize count() by bin(timestamp, 1m)
```

### GCP Cloud Monitoring Dashboard

```bash
# Use gcloud CLI
gcloud monitoring dashboards create --config-from-file=dashboard.json
```

**dashboard.json:**
```json
{
  "displayName": "LLM Monitoring",
  "dashboardFilters": [],
  "gridLayout": {
    "widgets": [
      {
        "title": "Request Latency",
        "xyChart": {
          "dataSets": [{
            "timeSeriesQuery": {
              "timeSeriesFilter": {
                "filter": "metric.type=\"custom.googleapis.com/llm_latency\""
              }
            }
          }]
        }
      },
      {
        "title": "Error Rate",
        "xyChart": {
          "dataSets": [{
            "timeSeriesQuery": {
              "timeSeriesFilter": {
                "filter": "metric.type=\"custom.googleapis.com/llm_errors\""
              }
            }
          }]
        }
      }
    ]
  }
}
```

---

## Setting Up Alerts

### AWS (SNS + CloudWatch)

```python
# Create SNS topic
sns = boto3.client('sns')
topic_response = sns.create_topic(Name='llm-alerts')
topic_arn = topic_response['TopicArn']

# Subscribe email
sns.subscribe(
    TopicArn=topic_arn,
    Protocol='email',
    Endpoint='devops@company.com'
)

# Create alarm
cloudwatch = boto3.client('cloudwatch')

cloudwatch.put_metric_alarm(
    AlarmName='HighLatency',
    MetricName='RequestLatency',
    Namespace='LLM-App',
    Statistic='Average',
    Period=300,  # 5 minutes
    EvaluationPeriods=1,
    Threshold=5000,  # 5 seconds in ms
    ComparisonOperator='GreaterThanThreshold',
    AlarmActions=[topic_arn]
)

cloudwatch.put_metric_alarm(
    AlarmName='HighErrorRate',
    MetricName='ErrorRate',
    Namespace='LLM-App',
    Statistic='Average',
    Period=300,
    EvaluationPeriods=2,
    Threshold=5,  # 5% error rate
    ComparisonOperator='GreaterThanThreshold',
    AlarmActions=[topic_arn]
)

cloudwatch.put_metric_alarm(
    AlarmName='DailyCostOverBudget',
    MetricName='DailyCost',
    Namespace='LLM-App',
    Statistic='Sum',
    Period=86400,  # 24 hours
    EvaluationPeriods=1,
    Threshold=500,  # $500/day
    ComparisonOperator='GreaterThanThreshold',
    AlarmActions=[topic_arn]
)
```

### Azure (Action Groups)

1. **Create Action Group**
   - Portal → Monitor → Alerts → Action Groups
   - Add action: Email, SMS, webhook, etc.

2. **Create Alert Rule**
   ```
   Condition: Application Insights Metric
   Metric: Server Response Time
   Operator: Greater than
   Threshold: 5000 ms
   Action Group: DevOps Team
   ```

### GCP (Notification Channels)

```bash
# Create notification channel
gcloud alpha monitoring channels create \
  --display-name="DevOps Email" \
  --type=email \
  --channel-labels=email_address=devops@company.com

# Create alert policy
gcloud alpha monitoring policies create \
  --notification-channels=CHANNEL_ID \
  --display-name="High Latency Alert" \
  --condition-display-name="Latency > 5s" \
  --condition-threshold-value=5000 \
  --condition-threshold-duration=300s
```

---

## Health Check Implementation

### Simple Health Check Endpoint

```python
import time
from datetime import datetime

@app.route('/health', methods=['GET'])
def health_check():
    """
    Returns health status of the LLM service
    Used by load balancers for auto-healing
    """
    try:
        # Check LLM connectivity (quick request)
        start = time.time()
        response = bedrock.invoke_model(
            modelId=MODEL_ID,
            body=json.dumps({
                "anthropic_version": "bedrock-2023-06-01",
                "max_tokens": 10,  # Minimal tokens for speed
                "messages": [{"role": "user", "content": "ok"}]
            })
        )
        latency = time.time() - start
        
        # Check database connectivity
        db_latency = check_db_connection()
        
        # Check cache connectivity
        cache_latency = check_cache_connection()
        
        # If all checks pass and latency is reasonable
        if latency < 2 and db_latency < 0.5 and cache_latency < 0.1:
            return jsonify({
                "status": "healthy",
                "timestamp": datetime.now().isoformat(),
                "latencies": {
                    "model": f"{latency*1000:.0f}ms",
                    "database": f"{db_latency*1000:.0f}ms",
                    "cache": f"{cache_latency*1000:.0f}ms"
                }
            }), 200
        else:
            return jsonify({"status": "degraded"}), 503
    
    except Exception as e:
        return jsonify({
            "status": "unhealthy",
            "error": str(e)
        }), 503

def check_db_connection():
    """Check database latency"""
    start = time.time()
    # Simple query
    connection.execute("SELECT 1")
    return time.time() - start

def check_cache_connection():
    """Check Redis/cache latency"""
    start = time.time()
    cache.ping()
    return time.time() - start
```

### Configure Load Balancer Health Checks

**AWS (ALB):**
```bash
# AWS CLI
aws elbv2 modify-target-group \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --health-check-path /health \
  --health-check-interval-seconds 30 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3
```

**Azure (Application Gateway):**
```
Settings → Backend settings → HTTP settings
Health probe: /health
Interval: 30 seconds
Timeout: 5 seconds
Unhealthy threshold: 3
```

**GCP (Cloud Load Balancing):**
```bash
gcloud compute health-checks create http llm-health-check \
  --request-path=/health \
  --port=8080 \
  --check-interval=30s \
  --timeout=5s
```

---

## Monitoring Checklist

- [ ] Set up availability monitoring
- [ ] Track latency at P50, P95, P99
- [ ] Monitor error rates
- [ ] Track token usage and costs
- [ ] Monitor GPU/resource utilization
- [ ] Create dashboards for daily viewing
- [ ] Set up alerts for:
  - [ ] High latency (>5 seconds)
  - [ ] High error rate (>1%)
  - [ ] Model unavailability
  - [ ] Cost overage
  - [ ] Resource limits
- [ ] Implement health check endpoints
- [ ] Configure load balancer health checks
- [ ] Set up on-call alerting
- [ ] Review metrics weekly

---

**Next:** See `06-Cost-Optimization/` for managing your LLM costs effectively.

---

**Key Takeaway:** Without monitoring, you won't know when things break. Start simple, add complexity as needed.
