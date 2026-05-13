# Kodosumi Runtime

Python-first runtime for deploying agents at scale. Built on Ray.

Marketplace concepts → [sokosumi-marketplace.md](sokosumi-marketplace.md).
MIP-003 agent shape → [agentic-services.md](agentic-services.md).
Payment integration → [masumi-payments.md](masumi-payments.md).

> Kodosumi-specific APIs below come from the official Kodosumi docs (https://docs.kodosumi.io). They are **not** independently verified against a live OpenAPI spec inside this skill — if a call fails, consult the live docs first.

---

## What Kodosumi Does

```
Your agent (any Python framework)
    ↓ deploy as Flow
Kodosumi Layer (access control · lifecycle · event streams · result aggregation)
    ↓ executes on
Ray distributed cluster (head + workers)
```

Wins:
- **Scale** — distributed across Ray workers, handles 100s+ concurrent jobs.
- **Lifecycle** — automated state transitions (queued → running → finished/error).
- **Events** — real-time stream per job.
- **Auth** — built-in API key + roles.
- **Framework-agnostic** — works with any Python AI framework.

---

## Concepts

| Term | Meaning |
|---|---|
| **Flow** | Your agent packaged for execution. Endpoint + entrypoint + config. |
| **Endpoint** | HTTP route that triggers the flow (e.g. `/my-agent`). |
| **Entrypoint** | Python callable (`module:function`) Kodosumi invokes. |
| **Ray head + workers** | Distributed compute; head schedules, workers execute. |
| **Spooler** | Event stream emitter; produces lifecycle events per job. |

---

## Ecosystem Position

```
Build → Deploy on Kodosumi → List on Sokosumi → Payments via Masumi
        (this file)            (sokosumi-marketplace.md)   (masumi-payments.md)
```

Use Kodosumi when:
- You need >100 concurrent jobs.
- Auto-scaling matters.
- You want managed Ray ops.

Skip Kodosumi when:
- Single-process FastAPI on Railway/Fly is enough.
- Your agent is bursty and small.

---

## Install + Configure

### Prereqs
- Python 3.9+
- Docker (optional)
- PostgreSQL

### Quick install
```bash
git clone https://github.com/masumi-network/kodosumi
cd kodosumi
pip install -r requirements.txt
cp .env.example .env
python scripts/init_db.py
python main.py
```

### Docker
```bash
docker-compose up -d        # dashboard at http://localhost:8000
```

### `.env`
```bash
# Ray
RAY_HEAD_ADDRESS=auto                # or "ip:port" to attach existing cluster
RAY_NUM_CPUS=4
RAY_NUM_GPUS=0

# API
API_HOST=0.0.0.0
API_PORT=8000
API_KEY=your-secret-api-key          # store in .env, NEVER inline

# DB
DATABASE_URL=postgresql://user:pass@localhost:5432/kodosumi

LOG_LEVEL=INFO
```

> `KODOSUMI_API_KEY` is **not** in the skill's standard env-var set (see [api-debug-recipes.md](api-debug-recipes.md)). If you build automation against Kodosumi, add it to your `.env` and follow the same safe-handling rules.

---

## Deploy a Flow

### 1. Structure your agent

```python
# my_agent/agent.py
class MyAgent:
    def __init__(self):
        pass                          # load models, configs

    def run(self, input_data: dict) -> dict:
        return {"status":"success","output": self.process(input_data)}

    def process(self, data):
        return f"Processed: {data}"

def entrypoint(input_data: dict) -> dict:
    return MyAgent().run(input_data)
```

### 2. `kodosumi_config.yaml`

```yaml
name: my-agent
version: "1.0.0"
description: "My AI agent on Kodosumi"

flow:
  endpoint: /my-agent
  entrypoint: my_agent.agent:entrypoint

resources:
  cpu: 2
  memory: 4GB
  gpu: 0

environment:
  OPENAI_API_KEY: ${OPENAI_API_KEY}   # interpolated from your shell/env, NEVER literal
  MODEL_NAME: gpt-4

dependencies:
  - openai>=1.0.0
  - langchain>=0.1.0
  - pydantic>=2.0.0
```

### 3. Deploy via CLI

```bash
pip install kodosumi-cli
kodosumi login --url https://your-kodosumi.com --api-key "$KODOSUMI_API_KEY"
kodosumi deploy my_agent/
kodosumi flows list
kodosumi flows invoke my-agent --data '{"task":"test"}'
```

> Pass the API key from env (`"$KODOSUMI_API_KEY"`). Never paste it literally.

### 4. Deploy via dashboard
Open `http://your-kodosumi.com` → Deploy New Flow → upload code + config.

---

## Flow Lifecycle

```
START → QUEUED → RUNNING → FINISHED
                    ↓
                  ERROR
```

| State | When |
|---|---|
| `START` | Invocation received |
| `QUEUED` | Waiting for worker |
| `RUNNING` | Executing on Ray worker |
| `FINISHED` | Completed successfully |
| `ERROR` | Failed |

### Monitor

```bash
curl -X GET "$KODOSUMI_URL/api/v1/flows/my-agent/jobs/job-123" \
  -H "Authorization: Bearer $KODOSUMI_API_KEY" | jq
```

Response includes `status` + `events[]` (timestamped lifecycle transitions).

Dashboard shows real-time updates + event stream + error logs.

---

## API Surface

> Verify against your Kodosumi instance's `/docs` swagger before relying on the exact shape — these are from the published docs.

| Method + Path | Purpose |
|---|---|
| `POST /api/v1/flows` | Deploy flow (multipart: name, code zip, config) |
| `POST /api/v1/flows/{name}/invoke` | Invoke flow with input data |
| `GET /api/v1/flows/{name}/jobs/{jobId}` | Job status + events |
| `GET /api/v1/flows` | List all flows |
| `DELETE /api/v1/flows/{name}` | Delete flow |

Auth: `Authorization: Bearer $KODOSUMI_API_KEY`.

### Invoke example
```bash
curl -X POST "$KODOSUMI_URL/api/v1/flows/my-agent/invoke" \
  -H "Authorization: Bearer $KODOSUMI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"input_data":{"task":"summarize"}}'
```

---

## Advanced Features

### File upload + processing
Flow accepts multipart `file` part; entrypoint receives a path/stream. Use for CSVs, PDFs, audio, etc.

### Resource locks
Mark entrypoint as exclusive on shared resource (GPU model, DB lock). Kodosumi serializes invocations holding that lock.

### Custom event streaming
Emit progress events from inside the entrypoint:
```python
from kodosumi import emit_event
emit_event(stage="loading_model", percentage=10)
emit_event(stage="inference", percentage=50)
```
Streamed via SSE on the job's events endpoint.

---

## Masumi Integration

### Payment-enabled flows
Declare in config so Kodosumi handles payment lifecycle:

```yaml
masumi:
  enabled: true
  payment_service_url: ${PAYMENT_SERVICE_URL}
  pricing:
    price_per_request: 10000000       # smallest unit (USDM)
    currency: usdm
```

Kodosumi will:
1. Receive invoke request.
2. Create payment request via Payment Service `POST /payment`.
3. Block flow execution until `FundsLocked`.
4. Run flow.
5. Auto-submit decision hash via `POST /payment/submit-result`.
6. Return result + `blockchainIdentifier`.

> Hash submission uses the verified body shape: `{network, blockchainIdentifier, submitResultHash}`. See [masumi-payments.md](masumi-payments.md).

### Registry integration
After deploy, register the Kodosumi endpoint URL as your agent's `apiBaseUrl` in `POST /registry` (Payment Service):

```bash
kodosumi flows list
# → https://your-kodo.com/api/v1/flows/my-agent
```

Use that URL when calling Payment Service `POST /registry` (full body shape → [registry-identity.md](registry-identity.md)).

---

## Best Practices

### Development
- Test the entrypoint as a plain Python function first.
- Local Ray (`ray start --head`) for end-to-end before deploying to remote cluster.
- Pin dependencies in `kodosumi_config.yaml`.

### Production
- Separate Kodosumi instances for preprod + mainnet.
- Set `resources.cpu / memory / gpu` honestly — Ray scheduler relies on it.
- Monitor worker count vs queue depth; scale Ray workers when queued > 0 persistently.
- PostgreSQL with backups + connection pooling.

### Performance
- Initialize heavy state (models, embeddings) **at module load** when worker boots, not inside `entrypoint()`.
- Use Ray actors for stateful agents that need cached context across invocations.
- Stream long outputs via `emit_event` instead of holding them in memory.

### Security
- `.env` keys only. Never bake API keys into the deployed code.
- Restrict who can deploy: separate `deploy` permission from `invoke`.
- HTTPS everywhere — Kodosumi endpoints must be HTTPS to be registered with Masumi.
- Rotate `KODOSUMI_API_KEY` regularly.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Flow stuck `QUEUED` | No idle worker; resource constraint not met | Scale Ray workers; lower flow's `resources.cpu` / `memory`. |
| `ERROR` immediately | Import error in entrypoint module | `kodosumi flows logs <name>` — check stack trace. |
| Slow first invocation | Worker booting + module import | Pre-warm: invoke a dummy at boot; or use Ray actor with init in `__init__`. |
| Masumi payment never locks | Wrong `payment_service_url`, network mismatch | Verify `PAYMENT_SERVICE_URL` reachable; `network` matches buyer's network. |
| Files not received | Wrong content-type, multipart field name | Use `multipart/form-data` with field name matching your entrypoint signature. |

---

## Resources

- Repo: https://github.com/masumi-network/kodosumi
- Docs: https://docs.kodosumi.io
- Ray: https://docs.ray.io

Next:
- Marketplace listing → [sokosumi-marketplace.md](sokosumi-marketplace.md)
- MIP-003 spec → [agentic-services.md](agentic-services.md)
- Payment integration → [masumi-payments.md](masumi-payments.md)
- Debug recipes → [api-debug-recipes.md](api-debug-recipes.md)
