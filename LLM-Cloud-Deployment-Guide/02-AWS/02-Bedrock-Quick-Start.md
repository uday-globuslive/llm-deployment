# AWS Bedrock - Quick Start Guide

## What You'll Learn

By the end of this, you'll have a working chatbot deployed on Lambda that uses Claude 3 Sonnet via Bedrock.

**Time Required:** 30 minutes
**Cost:** ~$0.10 to test

---

## Step 1: Enable Bedrock in Your AWS Account

### 1.1 Sign into AWS Console
- Go to https://console.aws.amazon.com
- Select region: **us-east-1** (Bedrock availability varies)

### 1.2 Request Model Access

**Why?** AWS makes you explicitly request access to each model to control costs.

1. Search for "Bedrock" in AWS Console
2. Click **Model Access** (left sidebar)
3. Click **Manage Model Access**
4. Under "Claude", check:
   - ✅ Claude 3 Sonnet
   - ✅ Claude 3 Haiku (cheaper alternative)
5. Click **Save changes**
6. Wait 1-2 minutes for approval (usually instant)

### 1.3 Verify Access

```bash
# In your terminal (AWS CLI must be installed)
aws bedrock list-foundation-models --region us-east-1

# Should see Claude models in the output
```

---

## Step 2: Create IAM User (Optional but Recommended)

**Why?** Avoid using AWS root account

### 2.1 Create User
1. AWS Console → IAM → Users → Create User
2. Username: `bedrock-dev`
3. Enable console access
4. Create custom password

### 2.2 Create Access Key
1. Click on user → Security credentials
2. Create access key → CLI
3. Save Access Key ID and Secret Access Key

### 2.3 Attach Policy
1. Click "Add permissions" → Attach policies directly
2. Search "bedrock"
3. Attach: **AmazonBedrockFullAccess**

---

## Step 3: Set Up Your Local Environment

### 3.1 Install AWS CLI

```bash
# Windows PowerShell
# Download and install from: https://aws.amazon.com/cli/
# Or use Chocolatey:
choco install awscli

# Verify
aws --version
# Output: aws-cli/2.x.x
```

### 3.2 Configure AWS Credentials

```bash
aws configure

# Enter your details:
# AWS Access Key ID: AKIA...
# AWS Secret Access Key: xxx...
# Default region: us-east-1
# Default output format: json
```

### 3.3 Install Python Dependencies

```bash
pip install boto3 flask
```

---

## Step 4: Create Your Application

### 4.1 Create `app.py`

```python
import boto3
import json
import os
from flask import Flask, request, jsonify

app = Flask(__name__)

# Initialize Bedrock client
bedrock = boto3.client('bedrock-runtime', region_name='us-east-1')

MODEL_ID = "anthropic.claude-3-sonnet-20240229-v1:0"

@app.route('/health', methods=['GET'])
def health():
    """Health check endpoint"""
    return jsonify({"status": "healthy"}), 200

@app.route('/chat', methods=['POST'])
def chat():
    """Chat endpoint - send prompt, get response"""
    try:
        data = request.json
        prompt = data.get('prompt', '')
        
        if not prompt:
            return jsonify({"error": "No prompt provided"}), 400
        
        # Call Bedrock
        response = bedrock.invoke_model(
            modelId=MODEL_ID,
            contentType='application/json',
            accept='application/json',
            body=json.dumps({
                "anthropic_version": "bedrock-2023-06-01",
                "max_tokens": 1024,
                "messages": [
                    {
                        "role": "user",
                        "content": prompt
                    }
                ]
            })
        )
        
        # Parse response
        result = json.loads(response['body'].read())
        response_text = result['content'][0]['text']
        
        return jsonify({
            "prompt": prompt,
            "response": response_text,
            "model": MODEL_ID,
            "usage": {
                "input_tokens": result['usage']['input_tokens'],
                "output_tokens": result['usage']['output_tokens']
            }
        }), 200
    
    except Exception as e:
        return jsonify({"error": str(e)}), 500

if __name__ == '__main__':
    app.run(debug=True, port=5000)
```

### 4.2 Test Locally

```bash
# Terminal 1: Run Flask app
python app.py
# Output: Running on http://127.0.0.1:5000

# Terminal 2: Send test request
curl -X POST http://localhost:5000/chat \
  -H "Content-Type: application/json" \
  -d '{"prompt": "What is DevOps?"}'

# Response:
# {
#   "response": "DevOps is a software development practice..."
# }
```

**Congratulations!** Your app is working locally.

---

## Step 5: Deploy to AWS Lambda

**Why Lambda?** Serverless, auto-scales, only pay when called

### 5.1 Create Lambda Deployment Package

```bash
# Create directory
mkdir lambda-deployment
cd lambda-deployment

# Copy app code
cp app.py lambda_function.py

# Edit lambda_function.py - change Flask to Lambda format
```

### 5.2 Modified Lambda Version (`lambda_function.py`)

```python
import boto3
import json

bedrock = boto3.client('bedrock-runtime', region_name='us-east-1')
MODEL_ID = "anthropic.claude-3-sonnet-20240229-v1:0"

def lambda_handler(event, context):
    """Lambda handler for Bedrock invocation"""
    try:
        # Parse input
        body = json.loads(event.get('body', '{}'))
        prompt = body.get('prompt', '')
        
        if not prompt:
            return {
                'statusCode': 400,
                'body': json.dumps({'error': 'No prompt provided'})
            }
        
        # Call Bedrock
        response = bedrock.invoke_model(
            modelId=MODEL_ID,
            contentType='application/json',
            accept='application/json',
            body=json.dumps({
                "anthropic_version": "bedrock-2023-06-01",
                "max_tokens": 1024,
                "messages": [{"role": "user", "content": prompt}]
            })
        )
        
        result = json.loads(response['body'].read())
        response_text = result['content'][0]['text']
        
        return {
            'statusCode': 200,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps({
                'response': response_text,
                'model': MODEL_ID,
                'tokens_used': {
                    'input': result['usage']['input_tokens'],
                    'output': result['usage']['output_tokens']
                }
            })
        }
    
    except Exception as e:
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }
```

### 5.3 Create Deployment Package

```bash
# Create zip file with code (no dependencies needed - boto3 built-in)
Compress-Archive -Path lambda_function.py -DestinationPath lambda_function.zip

# Verify
Expand-Archive lambda_function.zip -DestinationPath test-extract
```

### 5.4 Deploy to Lambda via AWS Console

1. **AWS Console → Lambda → Create Function**
   - Function name: `bedrock-chat`
   - Runtime: Python 3.12
   - Click "Create Function"

2. **Upload Code**
   - Code source: Upload from .zip file
   - Select `lambda_function.zip`
   - Click Deploy

3. **Set Timeout**
   - Configuration → General settings
   - Timeout: 60 seconds (Bedrock can be slow)
   - Save

4. **Test**
   - Test tab
   - Create test event:
   ```json
   {
     "body": "{\"prompt\": \"What is DevOps?\"}"
   }
   ```
   - Click Test
   - Should see response!

---

## Step 6: Create API Gateway

**Why?** Make Lambda callable via HTTPS REST API

### 6.1 Create API Gateway

1. **AWS Console → API Gateway → Create API**
   - Choose REST API
   - Click "Build"
   - API name: `bedrock-chat-api`
   - Click Create API

### 6.2 Create POST Endpoint

1. **Resources → Create Resource**
   - Resource name: `chat`
   - Click Create

2. **Create Method**
   - Select `/chat` → POST
   - Integration type: Lambda function
   - Lambda function: `bedrock-chat`
   - Click Save

3. **Enable CORS** (if calling from web)
   - Select `/chat` → Enable CORS
   - Click Save

### 6.3 Deploy API

1. **Deploy API**
   - Click "Deploy"
   - Stage: `dev`
   - Click Deploy

2. **Get Your Endpoint**
   - Look for "Invoke URL"
   - Copy it (looks like: https://xxx.execute-api.us-east-1.amazonaws.com/dev)

---

## Step 7: Test Your API

### 7.1 Test with Curl

```bash
# In PowerShell
$response = Invoke-WebRequest -Uri "https://your-api-url/chat" `
  -Method POST `
  -Headers @{"Content-Type"="application/json"} `
  -Body '{"prompt":"What is AWS Lambda?"}'

$response.Content

# Should see JSON response with your answer
```

### 7.2 Test with Python

```python
import requests
import json

url = "https://your-api-url/chat"
payload = {"prompt": "What is DevOps?"}

response = requests.post(url, json=payload)
result = response.json()

print(f"Response: {result['response']}")
print(f"Tokens used: {result['tokens_used']}")
```

---

## Step 8: Monitor and Track Costs

### 8.1 View Invocation Metrics

```bash
# CloudWatch Metrics
# AWS Console → CloudWatch → Dashboards
# Create dashboard showing:
# - Lambda invocations
# - Bedrock model invocations
# - Errors
```

### 8.2 Track Bedrock Costs

```bash
# AWS Console → Cost Management → Cost and Usage Reports
# Filter by:
# - Service: Amazon Bedrock
# - Model: Claude 3 Sonnet
# - Region: us-east-1
```

**Current Pricing (as of Jan 2026):**
- Input: $0.003 per 1K tokens
- Output: $0.015 per 1K tokens

**Example: 100 API calls with 100-token input, 200-token output**
- Cost = (100 × 100 × $0.003 / 1000) + (100 × 200 × $0.015 / 1000)
- Cost = $0.03 + $0.30 = $0.33

---

## Troubleshooting

### Issue: "Model access denied"
```bash
# Solution: Request model access via AWS Console
# AWS Console → Bedrock → Model Access → Manage → Check Claude
```

### Issue: "User is not authorized to perform"
```bash
# Solution: Check IAM permissions
# Attach AmazonBedrockFullAccess to your user
```

### Issue: Slow response (>10 seconds)
```bash
# Normal for Bedrock (first invocation slower)
# Lambda cold start + Bedrock processing time
# Solution: Use Provisioned Throughput for faster responses
```

### Issue: "Invalid model ID"
```bash
# Solution: Check MODEL_ID matches available models
aws bedrock list-foundation-models --region us-east-1 --output text
```

---

## Production Considerations

### 1. Add Error Handling
```python
try:
    # Bedrock call
except bedrock.exceptions.ValidationException as e:
    # Handle invalid input
except bedrock.exceptions.ThrottlingException as e:
    # Handle rate limits (add retry logic)
except Exception as e:
    # Handle other errors
```

### 2. Add Logging
```python
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    logger.info(f"Received prompt: {prompt}")
    # ...
    logger.info(f"Tokens used: {tokens}")
```

### 3. Add Input Validation
```python
if len(prompt) > 10000:
    return {"error": "Prompt too long"}

if not prompt.strip():
    return {"error": "Empty prompt"}
```

### 4. Add Rate Limiting
```python
# Store in DynamoDB or ElastiCache
# Limit to X requests per user per minute
```

### 5. Use Provisioned Throughput (if needed)
```bash
# AWS Console → Bedrock → Provisioned Throughput
# Set to: 100 tokens/minute (example)
# Guarantees consistent performance
# Cost: ~$4,000/month for 100 tokens/min
```

---

## Next Steps

1. **Want to optimize costs?** → See `06-Cost-Optimization/`
2. **Want monitoring/logging?** → See `05-Monitoring-and-Observability/`
3. **Want to scale to SageMaker?** → See `02-SageMaker-Production.md`
4. **Want multi-model setup?** → See `01-AWS-Options-Overview.md`

---

## Summary

You now have:
- ✅ Bedrock enabled in AWS
- ✅ Lambda function calling Bedrock
- ✅ API Gateway endpoint to call your Lambda
- ✅ Ability to send prompts and get responses
- ✅ Cost tracking setup

**Total deployment time:** ~30 minutes
**Monthly cost:** Variable (pay per token)
**Scalability:** Unlimited (AWS handles it)

**Key Bedrock Advantages:**
- Minimal infrastructure management
- Pay-per-use pricing
- Built-in availability
- No GPU management
- Easy to get started

---

**Next: Read about monitoring your Bedrock deployment in `05-Monitoring-and-Observability/01-Bedrock-Monitoring.md`**
