# Substack-Claude Content Assistant

A production-ready AWS serverless application that integrates Claude AI with Substack for intelligent content creation. Built with AWS Lambda, API Gateway, S3, DynamoDB, and following the AWS Well-Architected Framework.

## 🎯 Project Overview

This application demonstrates:
- **Serverless architecture** with AWS Lambda (ARM64/Graviton2)
- **Infrastructure as Code** using AWS SAM
- **Production best practices** - error handling, logging, monitoring
- **Well-Architected principles** - security, reliability, performance
- **External API integration** - Anthropic Claude API with retry logic
- **Multi-tier storage** - S3 for content, DynamoDB for metadata

**Perfect for AWS L4 interview preparation** - demonstrates real-world production patterns.

## 📋 Table of Contents

- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Local Development](#local-development)
- [Deployment](#deployment)
- [Testing](#testing)
- [API Documentation](#api-documentation)
- [Cost Estimation](#cost-estimation)
- [Interview Talking Points](#interview-talking-points)
- [Troubleshooting](#troubleshooting)

## 🏗️ Architecture

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
       ▼
┌──────────────────┐
│  API Gateway     │  ← REST API with throttling
└────────┬─────────┘
         │
         ▼
┌─────────────────────┐
│  Lambda Function    │  ← ARM64, 512MB, 30s timeout
│  (Generate Content) │
└──────┬──────────────┘
       │
       ├──► Secrets Manager  (Anthropic API key)
       │
       ├──► Anthropic API     (Claude Sonnet 4)
       │
       ├──► S3 Bucket        (Draft storage)
       │
       └──► DynamoDB         (Metadata)
```

### Key Design Decisions

| Decision | Rationale | Interview Talking Point |
|----------|-----------|-------------------------|
| **Lambda ARM64** | 20% cost savings + better performance/$ | Cost optimization principle |
| **S3 + DynamoDB split** | Large content in S3, metadata for queries in DynamoDB | Cost-effective storage pattern |
| **Secrets Manager** | Better rotation, auditing than Parameter Store | Security best practice |
| **API Gateway throttling** | Protect against cost overruns | Operational excellence |
| **Exponential backoff** | Handle rate limits gracefully | Reliability pattern |
| **Structured logging** | CloudWatch Insights queries | Observability |

## 📦 Prerequisites

### Required Tools

```bash
# AWS CLI v2
aws --version  # Should be 2.x

# AWS SAM CLI
sam --version  # Should be 1.100+

# Python 3.12+
python3 --version

# Docker (for local testing)
docker --version
```

### AWS Permissions Required

Your IAM user/role needs:
- CloudFormation (create/update stacks)
- Lambda (create/update functions)
- API Gateway (create APIs)
- S3 (create buckets)
- DynamoDB (create tables)
- Secrets Manager (create/update secrets)
- IAM (create roles/policies)
- CloudWatch Logs (create log groups)

### External Services

- **Anthropic API key** - Get from [console.anthropic.com](https://console.anthropic.com)

## 🚀 Quick Start

### 1. Clone and Setup

```bash
# Clone the repository
git clone <your-repo-url>
cd substack-claude-assistant

# Create virtual environment
python3 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### 2. Configure Environment

Create `tests/env.json` for local testing:

```json
{
  "GenerateContentFunction": {
    "ANTHROPIC_SECRET_NAME": "dev/anthropic/api-key",
    "S3_BUCKET_NAME": "dev-substack-drafts-123456789012",
    "DYNAMODB_TABLE_NAME": "dev-draft-metadata",
    "ENVIRONMENT": "dev",
    "LOG_LEVEL": "INFO"
  }
}
```

### 3. Build and Deploy

```bash
# Build the application
sam build

# Deploy (guided first time)
sam deploy --guided

# Follow the prompts:
# - Stack Name: substack-claude-dev
# - AWS Region: us-east-1 (or your preference)
# - Parameter Environment: dev
# - Confirm changes before deploy: Y
# - Allow SAM CLI IAM role creation: Y
# - Save arguments to config file: Y
```

### 4. Configure Secrets

```bash
# Make scripts executable
chmod +x scripts/*.sh

# Add your Anthropic API key
./scripts/setup_secrets.sh dev sk-ant-api03-YOUR-KEY-HERE
```

### 5. Test the API

```bash
# Get your API endpoint from SAM output
API_ENDPOINT=$(aws cloudformation describe-stacks \
  --stack-name substack-claude-dev \
  --query 'Stacks[0].Outputs[?OutputKey==`ApiEndpoint`].OutputValue' \
  --output text)

# Test content generation
curl -X POST ${API_ENDPOINT}/generate \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Write about the future of serverless computing",
    "tone": "professional",
    "length": "medium",
    "content_type": "article"
  }'
```

## 💻 Local Development

### Test Lambda Locally

```bash
# Test content generation
./scripts/local_test.sh generate

# Start local API (runs on http://localhost:3000)
./scripts/local_test.sh api
```

### Run Unit Tests

```bash
# Install test dependencies
pip install -r requirements.txt

# Run tests with coverage
pytest tests/unit/ -v --cov=src

# Run specific test
pytest tests/unit/test_handlers.py::test_generate_content -v
```

### Code Quality

```bash
# Format code
black src/ tests/

# Lint code
flake8 src/ tests/

# Type checking
mypy src/
```

## 📚 API Documentation

### POST /generate

Generate content using Claude AI.

**Request:**
```json
{
  "prompt": "Article topic or outline",
  "tone": "professional|casual|technical|creative|persuasive",
  "length": "short|medium|long",
  "content_type": "article|blog_post|newsletter|essay",
  "additional_context": "Optional context (max 2000 chars)",
  "target_audience": "Optional audience description"
}
```

**Response (200 OK):**
```json
{
  "draft_id": "550e8400-e29b-41d4-a716-446655440000",
  "content": "# Generated Content...",
  "preview": "First 500 characters...",
  "metadata": {
    "word_count": 1247,
    "character_count": 7891,
    "model": "claude-sonnet-4-20250514",
    "usage": {
      "input_tokens": 150,
      "output_tokens": 1500
    }
  },
  "s3_key": "drafts/2024-12-23/550e8400-e29b-41d4-a716-446655440000.md",
  "status": "success",
  "created_at": "2024-12-23T10:30:00.000Z"
}
```

**Error Responses:**
- `400` - Invalid request (validation failed)
- `429` - Rate limited
- `500` - Internal error
- `503` - AWS service unavailable

### GET /drafts/{draft_id}

Retrieve a previously generated draft.

**Response (200 OK):**
```json
{
  "draft_id": "550e8400-e29b-41d4-a716-446655440000",
  "content": "# Complete content...",
  "metadata": {
    "created_at": "2024-12-23T10:30:00.000Z",
    "status": "draft",
    "word_count": 1247,
    ...
  },
  "status": "success"
}
```

## 💰 Cost Estimation

### AWS Costs (Free Tier)

| Service | Free Tier | Beyond Free Tier |
|---------|-----------|------------------|
| Lambda | 1M requests/month | $0.20 per 1M requests |
| API Gateway | 1M calls/month | $3.50 per 1M calls |
| S3 | 5GB storage | $0.023 per GB |
| DynamoDB | 25GB storage | On-demand pricing |
| CloudWatch | 5GB logs | $0.50 per GB |

### Claude API Costs

| Model | Input | Output | Typical Article Cost |
|-------|-------|--------|---------------------|
| Sonnet 4 | $3/1M tokens | $15/1M tokens | $0.03 - $0.05 |

**Monthly estimate for 100 articles:** ~$3-5 (Claude API only)

## 🎤 Interview Talking Points

### Architecture & Design

**Q: Why Lambda instead of ECS or EC2?**
> "Lambda is ideal for sporadic workloads like content generation. With pay-per-request pricing and automatic scaling, we avoid paying for idle capacity. For this use case, Lambda costs 70-80% less than running an ECS service 24/7."

**Q: Why separate S3 and DynamoDB?**
> "This follows the pattern of storing large objects in S3 and metadata in DynamoDB for queries. DynamoDB queries are fast and cheap for 'list all drafts' or 'get by status', while S3 is more cost-effective for large content storage. We could also add S3 Select if we need to query content directly."

**Q: How do you handle cold starts?**
> "We're using ARM64 (Graviton2) which has faster cold starts than x86. I'm monitoring P99 latency in CloudWatch and could add Provisioned Concurrency if cold starts become an issue. For this workload, occasional 1-2 second cold starts are acceptable."

### Security

**Q: How do you secure API keys?**
> "Never hardcode - we use Secrets Manager with IAM policy limiting access to specific Lambda execution roles. The API key is retrieved once per Lambda container and cached. Secrets Manager also supports automatic rotation and provides audit logs via CloudTrail."

**Q: What about API rate limiting?**
> "We implement defense in depth:
> 1. API Gateway throttling (50 burst, 10 sustained)
> 2. Exponential backoff in code for Anthropic rate limits
> 3. CloudWatch alarms for unusual traffic patterns"

### Reliability

**Q: How do you handle API failures?**
> "We implement the retry pattern with exponential backoff - 3 retries with 2^n second delays. Transient failures (500, 502, 503, 429) are retried, but client errors (400, 401) fail fast. Failed invocations go to a Dead Letter Queue for manual investigation."

**Q: What if S3 or DynamoDB fail?**
> "We handle ClientError exceptions and return 503 with Retry-After header, telling clients to retry. For S3, we could add cross-region replication for higher durability. For DynamoDB, enabling PITR provides point-in-time recovery."

### Performance

**Q: How did you size the Lambda?**
> "Started with 512MB based on AWS's guidance for typical workloads. I'm monitoring CloudWatch metrics for memory usage and duration. If consistently under 256MB, I'll reduce to save costs. If hitting timeout, I'll increase memory (which also increases CPU)."

**Q: What about long-running requests?**
> "Claude API typically responds in 10-30 seconds. Our 30-second timeout provides buffer. For longer articles, we could use Step Functions to orchestrate the workflow, or implement async processing with SQS."

### Cost Optimization

**Q: How do you control costs?**
> "Multiple strategies:
> 1. ARM64 for 20% Lambda savings
> 2. S3 lifecycle policies (move to IA after 30 days, delete after 90)
> 3. DynamoDB on-demand pricing (pay per request)
> 4. Short CloudWatch log retention (7 days in dev)
> 5. API throttling to prevent runaway costs
> 6. Cost allocation tags on all resources"

## 🔧 Troubleshooting

### Common Issues

#### 1. "Secret not found" Error

**Problem:** Lambda can't retrieve Anthropic API key

**Solution:**
```bash
# Verify secret exists
aws secretsmanager describe-secret --secret-id dev/anthropic/api-key

# Update secret value
./scripts/setup_secrets.sh dev sk-ant-api03-YOUR-KEY

# Check Lambda IAM permissions
aws iam get-role-policy --role-name <lambda-execution-role> --policy-name <policy-name>
```

#### 2. "Access Denied" to S3/DynamoDB

**Problem:** Lambda lacks permissions

**Solution:**
- Check CloudFormation stack for correct IAM policies
- Verify resource names match environment variables
- Check CloudWatch Logs for specific error details

#### 3. Lambda Timeout

**Problem:** Execution exceeds 30 seconds

**Solution:**
```yaml
# In template.yaml, increase timeout
Timeout: 60  # Or 120 for long articles
```

#### 4. High Costs

**Problem:** Unexpected AWS bill

**Solution:**
```bash
# Check CloudWatch metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Invocations \
  --dimensions Name=FunctionName,Value=dev-generate-content \
  --start-time 2024-12-01T00:00:00Z \
  --end-time 2024-12-31T23:59:59Z \
  --period 86400 \
  --statistics Sum

# Review API Gateway throttling settings
# Enable AWS Cost Anomaly Detection
```

### Debugging Tips

1. **Check CloudWatch Logs**
   ```bash
   sam logs -n GenerateContentFunction --stack-name substack-claude-dev --tail
   ```

2. **Test Locally First**
   ```bash
   ./scripts/local_test.sh generate
   ```

3. **Validate SAM Template**
   ```bash
   sam validate --lint
   ```

4. **Check DynamoDB Items**
   ```bash
   aws dynamodb scan --table-name dev-draft-metadata --limit 5
   ```

## 📖 Next Steps (Phase 2)

- [ ] Add Substack API integration for publishing
- [ ] Implement conversation memory for multi-turn refinement
- [ ] Create simple web UI (S3 + CloudFront)
- [ ] Add content templates and style presets
- [ ] Implement draft versioning
- [ ] Add comprehensive test coverage
- [ ] Set up CI/CD pipeline (GitHub Actions)
- [ ] Create CloudWatch dashboard

## 🤝 Contributing

This is a learning project, but PRs welcome for:
- Bug fixes
- Documentation improvements
- Test coverage
- Performance optimizations

## 📝 License

MIT License - see LICENSE file for details

## 🙏 Acknowledgments

- Built following [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- Uses [Anthropic Claude API](https://www.anthropic.com/api)
- Inspired by real-world production patterns

---

**Built with ❤️ for AWS Cloud Institute students preparing for L4 interviews**
