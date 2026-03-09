# Kodosumi Runtime - Scalable Agent Deployment

Complete guide to deploying and managing AI agents at scale using Kodosumi, the Ray-based distributed execution runtime.

## Overview

**Kodosumi** is a Python-first runtime environment for deploying and executing agentic services at scale. Built on [Ray](https://ray.io), it provides distributed computing capabilities with managed lifecycle, event streaming, and seamless integration with the Masumi ecosystem.

### What is Kodosumi?

Kodosumi enables:
- **Scalable Deployment** - Deploy agents across distributed compute clusters
- **Lifecycle Management** - Automated flow execution from start to finish/error
- **Event Streaming** - Real-time job tracking and result aggregation
- **Access Control** - Built-in authentication and authorization
- **Framework Agnostic** - Works with any Python-based AI framework

```
┌──────────────────────────────────────────────────────┐
│               Your Agentic Service                    │
│    (CrewAI, AutoGen, LangGraph, Custom Python)       │
└────────────────────┬─────────────────────────────────┘
                     │
                     │ Deployed as Flow
                     ▼
┌──────────────────────────────────────────────────────┐
│                  Kodosumi Layer                       │
│  ┌────────────────────────────────────────────────┐  │
│  │  Access Control • Flow Control • Lifecycle     │  │
│  │  Event Streaming • Result Aggregation          │  │
│  └────────────────────────────────────────────────┘  │
└────────────────────┬─────────────────────────────────┘
                     │
                     │ Executes on
                     ▼
┌──────────────────────────────────────────────────────┐
│              Ray Distributed Cluster                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐     │
│  │ Head Node  │  │  Worker 1  │  │  Worker N  │     │
│  └────────────┘  └────────────┘  └────────────┘     │
└──────────────────────────────────────────────────────┘
```

## Key Concepts

### Flow
A **Flow** is your agentic service packaged for execution on Kodosumi. It consists of:
- **Endpoint**: HTTP route that triggers the flow (e.g., `/my-agent`)
- **Entrypoint**: Python callable that executes your agent logic
- **Configuration**: Environment variables, dependencies, resource requirements

### Endpoint
An **Endpoint** is the HTTP route that clients use to invoke your flow. Kodosumi exposes this via its API:
```
POST https://your-kodosumi-instance.com/api/v1/flows/my-agent
```

### Entrypoint
The **Entrypoint** is a Python function or method that Kodosumi calls to execute your flow:
```python
def my_agent_entrypoint(input_data: dict) -> dict:
    # Your agent logic here
    result = run_my_agent(input_data)
    return result
```

### Ray Head & Workers
- **Ray Head Node**: Coordinates job scheduling and orchestration
- **Ray Worker Nodes**: Execute the actual computation in parallel
- Kodosumi manages the Ray cluster automatically

### Spooler & Event Stream
- **Spooler**: Collects flow execution results and outputs
- **Event Stream**: Real-time feed of execution events (starting, progress, finished, error)

## Architecture

Kodosumi consists of three main building blocks:

1. **Your Service** - Any Python-based agentic application
2. **Kodosumi** - Manages lifecycle, events, and API layer
3. **Ray Cluster** - Distributed compute for execution at scale

### Integration with Masumi Ecosystem

```
┌─────────────────────────────────────────────────────┐
│  Sokosumi Marketplace                                │
│  (Agent Discovery & Job Management)                  │
└────────────────┬────────────────────────────────────┘
                 │
                 │ Discovers & Invokes
                 ▼
┌─────────────────────────────────────────────────────┐
│  Kodosumi Runtime                                    │
│  (Scalable Execution)                                │
└────────────────┬────────────────────────────────────┘
                 │
                 │ Handles Payments
                 ▼
┌─────────────────────────────────────────────────────┐
│  Masumi Protocol                                     │
│  (Blockchain Payments & Identity)                    │
└─────────────────────────────────────────────────────┘
```

**Typical workflow:**
1. Build agent with any framework
2. Deploy as Flow on Kodosumi
3. List on Sokosumi marketplace
4. Masumi handles payments when agent is hired

## Getting Started

### Installation

**Prerequisites:**
- Python 3.9+
- Docker (optional, for containerized deployment)
- PostgreSQL (for persistent storage)

**Quick Install:**
```bash
# Clone Kodosumi repository
git clone https://github.com/masumi-network/kodosumi
cd kodosumi

# Install dependencies
pip install -r requirements.txt

# Set up configuration
cp .env.example .env
# Edit .env with your settings

# Initialize database
python scripts/init_db.py

# Start Kodosumi
python main.py
```

**Docker Setup:**
```bash
# Build and run with Docker Compose
docker-compose up -d

# Access dashboard at http://localhost:8000
```

### Configuration

Key environment variables in `.env`:

```bash
# Ray Configuration
RAY_HEAD_ADDRESS=auto  # or specific IP for existing cluster
RAY_NUM_CPUS=4
RAY_NUM_GPUS=0

# Kodosumi API
API_HOST=0.0.0.0
API_PORT=8000
API_KEY=your-secret-api-key

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/kodosumi

# Logging
LOG_LEVEL=INFO
```

## Deploying Your Agent

### Step 1: Prepare Your Agent

Structure your agent as a Python module:

```python
# my_agent/agent.py
from typing import Dict, Any

class MyAgent:
    def __init__(self):
        # Initialize your agent (load models, configs, etc.)
        pass

    def run(self, input_data: Dict[str, Any]) -> Dict[str, Any]:
        # Your agent logic
        result = self.process(input_data)
        return {
            "status": "success",
            "output": result
        }

    def process(self, data):
        # Actual processing logic
        return f"Processed: {data}"

# Entrypoint function for Kodosumi
def entrypoint(input_data: dict) -> dict:
    agent = MyAgent()
    return agent.run(input_data)
```

### Step 2: Create Deployment Configuration

Create `kodosumi_config.yaml`:

```yaml
name: my-agent
version: "1.0.0"
description: "My AI agent deployed on Kodosumi"

flow:
  endpoint: /my-agent
  entrypoint: my_agent.agent:entrypoint

resources:
  cpu: 2
  memory: 4GB
  gpu: 0

environment:
  OPENAI_API_KEY: ${OPENAI_API_KEY}
  MODEL_NAME: gpt-4

dependencies:
  - openai>=1.0.0
  - langchain>=0.1.0
  - pydantic>=2.0.0
```

### Step 3: Deploy via CLI

```bash
# Install Kodosumi CLI
pip install kodosumi-cli

# Login to your Kodosumi instance
kodosumi login --url https://your-kodosumi.com --api-key YOUR_API_KEY

# Deploy your flow
kodosumi deploy my_agent/

# Verify deployment
kodosumi flows list

# Test your flow
kodosumi flows invoke my-agent --data '{"task": "test"}'
```

### Step 4: Deploy via Dashboard

1. Navigate to Kodosumi dashboard at `http://your-kodosumi.com`
2. Click "Deploy New Flow"
3. Upload your agent code and configuration
4. Click "Deploy"
5. Monitor deployment status in real-time

## Flow Lifecycle

Kodosumi manages the complete lifecycle of your flow:

```
START → QUEUED → RUNNING → FINISHED
                    ↓
                  ERROR
```

**Lifecycle States:**
- **START**: Flow invocation received
- **QUEUED**: Waiting for available worker
- **RUNNING**: Executing on Ray worker
- **FINISHED**: Completed successfully
- **ERROR**: Failed with error details

### Monitoring Flow Execution

**Via API:**
```bash
# Get flow status
curl -X GET https://your-kodosumi.com/api/v1/flows/my-agent/jobs/job-123 \
  -H "Authorization: Bearer YOUR_API_KEY"

# Response
{
  "job_id": "job-123",
  "flow_name": "my-agent",
  "status": "RUNNING",
  "created_at": "2025-03-09T10:00:00Z",
  "events": [
    {"timestamp": "2025-03-09T10:00:00Z", "event": "START"},
    {"timestamp": "2025-03-09T10:00:01Z", "event": "QUEUED"},
    {"timestamp": "2025-03-09T10:00:05Z", "event": "RUNNING"}
  ]
}
```

**Via Dashboard:**
- Real-time job status updates
- Event stream visualization
- Error logs and debugging info

## API Reference

### Deploy Flow
```http
POST /api/v1/flows
Authorization: Bearer {API_KEY}
Content-Type: multipart/form-data

{
  "name": "my-agent",
  "code": <file>,
  "config": <kodosumi_config.yaml>
}
```

### Invoke Flow
```http
POST /api/v1/flows/{flow_name}/invoke
Authorization: Bearer {API_KEY}
Content-Type: application/json

{
  "input_data": {
    "task": "analyze data"
  }
}
```

### Get Flow Status
```http
GET /api/v1/flows/{flow_name}/jobs/{job_id}
Authorization: Bearer {API_KEY}
```

### List All Flows
```http
GET /api/v1/flows
Authorization: Bearer {API_KEY}
```

### Delete Flow
```http
DELETE /api/v1/flows/{flow_name}
Authorization: Bearer {API_KEY}
```

## Advanced Features

### File Upload & Processing

Kodosumi supports file uploads for flows that need to process documents, images, etc.:

```python
def entrypoint(input_data: dict, files: list = None) -> dict:
    if files:
        for file in files:
            # Process uploaded files
            content = file.read()
            # Your processing logic

    return {"status": "success"}
```

**Invoke with files:**
```bash
curl -X POST https://your-kodosumi.com/api/v1/flows/my-agent/invoke \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -F "input_data={\"task\": \"analyze\"}" \
  -F "file=@document.pdf"
```

### Resource Locks

Prevent concurrent execution when needed:

```yaml
flow:
  endpoint: /my-agent
  entrypoint: my_agent.agent:entrypoint
  locks:
    - resource_name: "database_access"
      max_concurrent: 1
```

### Custom Event Streaming

Emit custom events during execution:

```python
from kodosumi import emit_event

def entrypoint(input_data: dict) -> dict:
    emit_event("Processing started")

    result = process_data(input_data)
    emit_event(f"Processed {len(result)} items")

    emit_event("Finalizing output")
    return {"result": result}
```

## Integration with Masumi

### Payment-Enabled Flows

Integrate with Masumi for automatic payment handling:

```yaml
name: my-paid-agent
version: "1.0.0"

masumi:
  enabled: true
  payment_service_url: https://your-masumi-node.com
  api_key: ${MASUMI_API_KEY}
  pricing:
    price_per_request: 10  # USDM
    currency: USDM

flow:
  endpoint: /my-paid-agent
  entrypoint: my_agent.agent:entrypoint
```

**Kodosumi will automatically:**
1. Verify payment before execution
2. Track decision logging hashes
3. Submit results to Masumi smart contract

### Registry Integration

Register your Kodosumi-deployed agent on Masumi registry:

```bash
# After deploying on Kodosumi
masumi-cli register \
  --name "My Agent" \
  --description "AI agent deployed on Kodosumi" \
  --api-endpoint "https://your-kodosumi.com/api/v1/flows/my-agent/invoke" \
  --price 10
```

## Documentation Resources

**Official Kodosumi Docs:**
- Homepage: https://docs.kodosumi.io
- [What is Kodosumi?](https://docs.kodosumi.io/guides/what-is-kodosumi.md)
- [Installation Guide](https://docs.kodosumi.io/guides/installation.md)
- [Deployment Guide](https://docs.kodosumi.io/guides/deploy.md)
- [Flow Lifecycle](https://docs.kodosumi.io/guides/lifecycle.md)
- [Configuration](https://docs.kodosumi.io/guides/config.md)
- [Dashboard Usage](https://docs.kodosumi.io/guides/dashboard.md)
- [File Upload](https://docs.kodosumi.io/guides/files.md)
- [CLI Reference](https://docs.kodosumi.io/cli.md)
- [API Reference](https://docs.kodosumi.io/api-reference.md)

## Best Practices

### Development Workflow
1. ✅ Develop locally first, test thoroughly
2. ✅ Use version control for your agent code
3. ✅ Test with small datasets before scaling
4. ✅ Monitor resource usage (CPU, memory, GPU)
5. ✅ Implement proper error handling

### Production Deployment
1. ✅ Use environment variables for secrets (never hardcode)
2. ✅ Set appropriate resource limits
3. ✅ Configure auto-scaling based on load
4. ✅ Enable logging and monitoring
5. ✅ Have rollback strategy

### Performance Optimization
1. ✅ Batch processing when possible
2. ✅ Cache frequently used data
3. ✅ Use GPU workers for ML models
4. ✅ Profile and optimize bottlenecks
5. ✅ Consider async/parallel execution

### Security
1. ✅ Rotate API keys regularly
2. ✅ Use HTTPS for all endpoints
3. ✅ Validate all input data
4. ✅ Implement rate limiting
5. ✅ Audit access logs

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Flow won't deploy | Check Python dependencies, verify entrypoint path |
| Execution timeout | Increase timeout in config, optimize agent code |
| Out of memory | Reduce batch size, increase worker memory |
| Ray cluster not starting | Check Ray head address, verify network connectivity |
| API authentication fails | Verify API key, check authorization header format |
| Event stream not updating | Check WebSocket connection, refresh dashboard |

## Support

- **Documentation**: https://docs.kodosumi.io
- **GitHub**: https://github.com/masumi-network/kodosumi
- **Issues**: https://github.com/masumi-network/kodosumi/issues
- **Email**: support@kodosumi.io

---

**Kodosumi powers the scalable execution layer of the Masumi ecosystem, enabling AI agents to serve users at any scale while maintaining reliability and performance.**
