# GCP Agentic AI — Interview Mastery Guide

> Written for a DevOps / SRE / MLOps lead preparing for senior agentic AI interviews.
> Every scenario reflects real daily work. Every service definition includes why you
> chose it and what problem it solves. Use these as talking points, not scripts.

---

## Part 1: GCP Agentic AI Services — Definitions & Why You Use Each

Understanding *why* you chose each service is the difference between a candidate who
read the docs and one who actually shipped it in production.

---

### 1. Gemini Enterprise Agent Platform (formerly Vertex AI Agent Builder)

**What it is:**
Google's unified platform for building, deploying, and governing AI agents in production.
Announced as a rebrand at Cloud Next 2026. Bundles ADK, Agent Studio (low-code), Agent
Engine (managed runtime), RAG Engine, Memory Bank, and 200+ models under one roof.

**Why you use it:**
You don't want to stitch together separate ML services, vector DBs, and hosting infra
yourself. This gives you the full stack — model serving, retrieval, agent runtime,
observability — on a single managed platform with IAM baked in.

**When to use vs alternatives:**
Use this when the primary backbone is GCP. If you're multi-cloud or need framework
flexibility, you run ADK on GKE instead.

---

### 2. Agent Development Kit (ADK)

**What it is:**
Google's open-source, code-first Python/Go/Java/TypeScript framework for building agents.
Released GA at Cloud Next 2026. It is the same framework that powers Google's own
Agentspace and Customer Engagement Suite. Supports multi-agent orchestration patterns:
sequential, parallel, hierarchical, and loop-based.

**Why you use it:**
ADK gives you production-grade control over agent logic that drag-and-drop tools cannot.
You define tools, manage state, write routing logic, and plug in any model — Gemini,
Claude, or open-source via Ollama. When you need custom retry logic, conditional branching,
or sub-agent delegation, ADK is the only real choice on GCP.

**Key ADK patterns:**
- `LlmAgent` — the base agent class backed by an LLM
- `SequentialAgent` — runs sub-agents in order, passes context through
- `ParallelAgent` — runs sub-agents concurrently, merges results
- `LoopAgent` — runs until a termination condition is met (useful for polling or retries)
- Tools can be Python functions, MCP servers, OpenAPI endpoints, or other agents

**Deployment:** ADK agents deploy to Agent Engine (fully managed) or GKE (self-managed).

---

### 3. Agent Engine (formerly Vertex AI Agent Engine, also called "Deployments")

**What it is:**
The managed serverless runtime for deploying ADK agents. Handles auto-scaling, session
management, health checks, and observability without you running any Kubernetes for it.
Supports sub-second cold starts and long-running agentic workflows.

**Why you use it:**
You don't want to manage pod scaling for bursty agent workloads. Agent Engine abstracts
the infra. You deploy a packaged ADK agent and get a REST endpoint. It integrates with
Cloud Monitoring, Cloud Logging, and Cloud Trace out of the box.

**Use case:** Any agent that needs to handle variable load, maintain session state,
and serve production traffic with SLAs.

---

### 4. A2A — Agent2Agent Protocol

**What it is:**
An open, HTTP-based interoperability protocol (now governed by the Linux Foundation) that
lets AI agents built on *different platforms* communicate with each other without sharing
internal architecture or credentials. As of June 2026, 150+ organisations use it in
production, including Salesforce, ServiceNow, SAP, and Deloitte.

**Core concepts:**
- **Agent Card** (`/.well-known/agent.json`): A JSON manifest describing what the agent
  can do, its skills, supported input/output modes, and authentication requirements.
  Think of it as an OpenAPI spec but for agents.
- **Task**: The unit of work — an agent sends a task to another agent and polls for its
  result or streams it via SSE.
- **Transport**: Built on standard HTTP + JSON-RPC + Server-Sent Events (SSE). No
  proprietary binary protocol. Any language can implement it.
- **Auth**: OAuth 2.0 + JWT. Agents authenticate each other without passing credentials.

**Why you use it:**
In enterprise environments you never own every system. Your GCP agent needs to hand off
a task to a ServiceNow agent your ITSM team manages, or delegate enrichment to a Salesforce
agent the CRM team owns. A2A is how you do that without building custom integrations for
every pair of agents.

**Example in daily work:**
Your DevOps orchestrator agent (on GCP) uses A2A to call a PagerDuty agent (external)
to create an incident, a Slack agent to notify the on-call, and a Confluence agent to
pull the runbook — all in one workflow, all authenticated, all observable.

---

### 5. RAG Engine

**What it is:**
A fully managed Retrieval-Augmented Generation service on the Gemini Enterprise Agent
Platform. It manages the full RAG pipeline: document ingestion, chunking, embedding
generation, vector storage (default: Spanner-backed RagManagedDB), retrieval, and
context injection into the LLM prompt.

**Why you use it:**
RAG solves hallucination at the data layer. When your agent answers a question about your
infrastructure, you don't want it making things up — you want it grounded in your actual
runbooks, architecture docs, and post-mortems. RAG Engine handles the messy parts
(embedding freshness, chunk sizing, hybrid search) so you focus on the application.

**When RAG vs fine-tuning:**
RAG is the right choice when your data changes frequently (runbooks, incident reports,
pricing data). Fine-tuning is for baking in style, tone, or domain reasoning that doesn't
change. In operations contexts, almost always start with RAG.

---

### 6. Vertex AI Vector Search

**What it is:**
Google's scalable, low-latency approximate nearest neighbour (ANN) search service.
Backs similarity search for RAG at enterprise scale. Built on the same tech as Google
Search's internal embedding retrieval.

**Why you use it over RAG Engine's built-in store:**
When you already have embeddings in your own pipeline (e.g., generated by a custom
fine-tuned encoder), or need to query millions of vectors with sub-100ms SLA, you bring
your own Vector Search index rather than using RAG Engine's managed store. Also useful
for multi-modal search (images + text).

---

### 7. Grounding (Google Search Grounding + RAG Grounding)

**What it is:**
Grounding connects a Gemini response to a verifiable source before returning it to the
user. Two types:
- **Google Search Grounding**: Gemini queries live Google Search to verify facts.
- **RAG Grounding**: Gemini queries your private vector store (via RAG Engine) to anchor
  responses in your internal documents.

**Why you use it:**
Without grounding, LLMs are confident liars. With grounding, every claim in the response
can be traced back to a source document. In regulated environments (finance, healthcare,
SOC 2-audited orgs), you *cannot* deploy LLMs without grounding — an auditor will ask
where the response came from.

---

### 8. Memory Bank

**What it is:**
A managed, persistent key-value + semantic memory store for agents. Stores long-term
context across sessions so your agent remembers that last week you told it your GKE cluster
uses node auto-provisioning, and doesn't ask again this week.

**Why you use it:**
Stateless agents are frustrating in daily operational use. If your DevOps assistant
forgets the team's infra context every conversation, engineers stop using it.
Memory Bank gives agents continuity across sessions without you building a custom
state management layer.

**Use case:** Storing user preferences, past decisions, infrastructure inventory
snapshots, and team-specific context that improves agent responses over time.

---

### 9. MCP — Model Context Protocol

**What it is:**
An open standard (created by Anthropic, adopted by Google) for exposing tools and
data sources to LLMs in a standardised way. An MCP server exposes "tools" (functions
the LLM can call) over a well-known interface. ADK supports MCP natively.

**Pre-built GCP MCP servers:** BigQuery, Google Maps, Cloud Logging, Gmail, Google Drive.

**Why you use it:**
Instead of writing custom tool wrappers for every API your agent needs to call, you
run an MCP server and the agent discovers its tools automatically. Your BigQuery MCP
server means your cost-analysis agent can query your billing dataset with a single tool
call without you hardcoding the query interface.

---

### 10. LangGraph (third-party, used on GCP)

**What it is:**
A Python framework from LangChain for building stateful, cyclic (non-DAG) agent graphs.
Lets you model complex agent workflows as state machines with explicit nodes, edges,
and conditional routing. ADK integrates LangGraph agents as sub-agents.

**Why you use it:**
When your workflow is not a simple sequence — when an agent needs to loop, retry,
branch based on intermediate results, or hold state across many tool calls — LangGraph's
explicit state machine model is safer than a free-form ReAct loop. You can draw the
workflow, test individual nodes, and trace execution step by step.

---

### 11. Cloud Run (for A2A agent hosting)

**What it is:**
Google's serverless container platform. Ideal for hosting lightweight A2A-compliant
agents that need to be called by other agents over HTTP.

**Why you use it:**
If you're building a specialised agent (a "skill agent" in A2A terms) that one
orchestrator calls occasionally, Cloud Run is cheaper and simpler than GKE. It
scales to zero, and an A2A agent card is just a static JSON file at a well-known path.

---

### 12. GKE (Google Kubernetes Engine) — for ADK at scale

**What it is:**
Google's managed Kubernetes service. When Agent Engine's managed runtime is too
constraining (custom networking, GPU-attached nodes, non-standard dependencies),
you deploy ADK agents on GKE yourself.

**Why you use it:**
Large multi-agent systems with GPU-backed model inference, complex sidecar patterns,
or strict network segmentation requirements need GKE. Agent Engine is sufficient for
most use cases; GKE is for when you've outgrown it.

---

### 13. Cloud Logging + Cloud Trace + Cloud Monitoring (Observability for Agents)

**Why this matters for agentic AI specifically:**
An agent that makes 12 tool calls before answering a question is a black box without
good tracing. Cloud Trace shows you the full span of every agent turn — which tool
was called, how long it took, and what it returned. Without this, debugging a misbehaving
agent is guesswork.

**What to instrument:**
- Every tool call: input, output, latency
- Every LLM call: model, prompt tokens, completion tokens, cost
- Agent memory reads/writes
- A2A call latencies across agent boundaries

---

## Part 2: The 3 Production Scenarios

---

## Scenario 1: SRE Incident Intelligence System (RAG + Grounding + ADK)

### The Story (tell this in interviews)

> "At Appfire, our SRE team was drowning in alert noise. We had 500+ runbooks,
> post-mortems, and architecture docs spread across Confluence, Google Drive, and
> GitHub. When a P1 came in at 3 AM, the on-call engineer spent 20 minutes searching
> docs before even starting to fix the issue. I built an Incident Intelligence agent
> that lets on-call engineers ask in plain English: 'Payments service is throwing 503s
> on the checkout endpoint, what do I do?' and it returns the relevant runbook steps,
> the last 3 similar incidents, and the escalation path — in under 5 seconds."

### Services Used & Why

| Service | Role | Why This One |
|---|---|---|
| **Gemini 2.5 Pro** | LLM backbone | Best at long-context reasoning over docs |
| **RAG Engine** | Private knowledge retrieval | Manages ingestion + retrieval without custom infra |
| **Cloud Storage (GCS)** | Document source | All runbooks/post-mortems stored as markdown/PDF |
| **ADK** | Agent orchestration | Custom pre/post processing logic around RAG call |
| **Grounding** | Source citation | On-call engineers need to see *which* doc the answer came from |
| **Agent Engine** | Serving | Managed runtime, auto-scales for bursty incident traffic |
| **Memory Bank** | Session continuity | Agent remembers current incident context across follow-up questions |
| **Cloud Logging** | Audit trail | SOC 2 requirement: log every response and its source |

### Step-by-Step Implementation

**Step 1: Ingest Documents into RAG Engine**

```python
import vertexai
from vertexai.preview import rag

vertexai.init(project="appfire-sre", location="us-central1")

# Create a RAG corpus — this is your knowledge base
corpus = rag.create_corpus(
    display_name="sre-runbooks-corpus",
    description="All runbooks, post-mortems, and architecture docs"
)

# Import from GCS — supports PDF, Markdown, HTML, DOCX
rag.import_files(
    corpus_name=corpus.name,
    paths=["gs://appfire-runbooks/", "gs://appfire-postmortems/"],
    chunk_size=512,        # tokens per chunk — tune based on doc structure
    chunk_overlap=100,     # overlap prevents context loss at chunk boundaries
    max_embedding_requests_per_min=900
)
```

**Why chunk_size=512?** Runbooks have dense, structured content. Smaller chunks
(256) lose inter-step context. Larger chunks (1024) dilute retrieval precision.
512 is the empirical sweet spot for procedural docs.

**Step 2: Build the ADK Agent with RAG Tool**

```python
from google.adk.agents import LlmAgent
from google.adk.tools.retrieval import VertexAiRagRetrieval
from vertexai.preview import rag

# Define the RAG retrieval tool
rag_tool = VertexAiRagRetrieval(
    name="sre_knowledge_base",
    description=(
        "Search SRE runbooks, past incident post-mortems, and architecture docs. "
        "Use this for any question about how to handle an incident or what a service does."
    ),
    rag_resources=[
        rag.RagResource(rag_corpus="projects/appfire-sre/locations/us-central1/ragCorpora/CORPUS_ID")
    ],
    similarity_top_k=5,           # return top 5 most relevant chunks
    vector_distance_threshold=0.7  # ignore chunks below this relevance score
)

# Build the agent
incident_agent = LlmAgent(
    name="sre_incident_agent",
    model="gemini-2.5-pro",
    instruction="""You are the SRE on-call assistant for Appfire.
    When an engineer reports an incident, always:
    1. Search the knowledge base for relevant runbooks and past incidents
    2. Cite the specific document and section you are drawing from
    3. Give step-by-step remediation in numbered list format
    4. Always include the escalation path at the end
    Never make up steps that are not in the documentation.
    If you cannot find a relevant runbook, say so explicitly.""",
    tools=[rag_tool],
    generate_content_config={
        "temperature": 0.1,  # low temp = more deterministic, safer for ops
        "max_output_tokens": 2048
    }
)
```

**Step 3: Add Memory Bank for session continuity**

```python
from google.adk.memory import VertexAiMemoryBankService
from google.adk.sessions import InMemorySessionService

memory_service = VertexAiMemoryBankService(
    project="appfire-sre",
    location="us-central1"
)

# Memory stores: current incident severity, affected services, steps already tried
# Next question in the session benefits from this context automatically
```

**Step 4: Deploy to Agent Engine**

```bash
# Package and deploy
gcloud ai agent-engines create \
  --display-name="sre-incident-agent" \
  --agent-framework=adk \
  --source-dir=./incident_agent/ \
  --requirements=requirements.txt \
  --region=us-central1
```

**Step 5: Wire up to Slack (the real daily-use interface)**

The agent gets a Cloud Run HTTP endpoint. Your Slack bot POSTs the engineer's message to
this endpoint and returns the formatted response in the incident channel. On-call engineers
never leave Slack to query the agent.

### Production Considerations

- **Freshness**: Runbooks change. Schedule a weekly Cloud Scheduler job to re-import
  updated GCS files so the corpus stays current.
- **Evaluation**: Run weekly evals using a golden set of 50 past incident questions
  with known correct answers. Track retrieval precision, answer relevance, citation accuracy.
- **Cost control**: Each RAG query costs ~$0.002–$0.005. At 200 queries/day, that's
  <$30/month. Log token counts per query; set budget alerts at $50/month.
- **Access control**: Corpus-level IAM — only the SRE service account can query it.
  Engineers query via the agent, never directly.

---

## Scenario 2: Multi-Agent CI/CD Review Pipeline (ADK + Multi-Agent + MCP + A2A)

### The Story (tell this in interviews)

> "Every PR at Appfire touching infrastructure had to be manually reviewed for security
> issues, cost impact, and SOC 2 compliance before merge. This was taking 2–3 hours of
> senior engineer time per PR. I built a multi-agent pipeline using ADK that runs
> automatically on every PR: a Security Agent scans for credential exposure and IAM
> over-permissioning, a Cost Agent estimates the monthly spend delta using GCP pricing
> data from BigQuery, and a Compliance Agent checks against our SOC 2 policy checklist.
> An Orchestrator agent runs them in parallel, merges their reports, and posts a
> structured review comment to GitHub — all within 90 seconds of the PR opening.
> We cut infrastructure PR review time from 3 hours to 15 minutes of human review."

### Services Used & Why

| Service | Role | Why This One |
|---|---|---|
| **ADK (ParallelAgent)** | Orchestration | Run 3 review agents concurrently, not sequentially |
| **Gemini 2.5 Flash** | Agent LLM | Flash is cheaper + faster; sufficient for structured analysis tasks |
| **BigQuery MCP Server** | Cost data | GCP billing export is in BigQuery — MCP tool lets agent query it directly |
| **GitHub MCP Server** | PR access | Reads PR diff, posts comments without custom GitHub API wrappers |
| **Agent Engine** | Runtime | Managed, auto-scales for PR webhook bursts |
| **Cloud Run** | Webhook receiver | Receives GitHub webhook, triggers agent asynchronously |
| **Secret Manager** | Credentials | GitHub token, GCP SA key never hardcoded in agent code |
| **Cloud Pub/Sub** | Decoupling | Webhook → Pub/Sub → Agent Engine (handles GitHub's 10s timeout) |
| **Cloud Trace** | Debugging | Full trace of which agent found what, how long each took |

### Multi-Agent Architecture

```
GitHub PR Opened
      │
      ▼
Cloud Run (webhook receiver)
      │ publishes message
      ▼
Cloud Pub/Sub topic: pr-review-requests
      │ triggers
      ▼
Orchestrator Agent (ADK ParallelAgent)
      │
      ├──────────────────────────────────────┐──────────────────────────────────────┐
      ▼                                      ▼                                      ▼
Security Agent                         Cost Agent                         Compliance Agent
(scans for exposed secrets,            (queries BigQuery billing,         (checks SOC 2 controls:
 IAM over-permission, public           estimates monthly delta            MFA required, no public
 buckets, hardcoded IPs)               from Terraform changes)            buckets, encryption at rest)
      │                                      │                                      │
      └──────────────────────────────────────┘──────────────────────────────────────┘
                                             │
                                             ▼
                                    Orchestrator merges reports
                                             │
                                             ▼
                              GitHub MCP → Post PR comment
```

### Step-by-Step Implementation

**Step 1: Define the three specialist agents**

```python
from google.adk.agents import LlmAgent, ParallelAgent
from google.adk.tools.mcp import MCPToolset, SseServerParams

# BigQuery MCP for cost queries
bq_mcp = MCPToolset(
    connection_params=SseServerParams(
        url="https://mcp-server-bigquery-xyz.run.app/sse"
    )
)

# GitHub MCP for PR access
github_mcp = MCPToolset(
    connection_params=SseServerParams(
        url="https://mcp-server-github-xyz.run.app/sse"
    )
)

security_agent = LlmAgent(
    name="security_reviewer",
    model="gemini-2.5-flash",
    instruction="""Review the provided Terraform and code diff for:
    1. Hardcoded secrets, API keys, passwords
    2. IAM roles broader than least-privilege (e.g., roles/owner, roles/editor)
    3. Public-facing storage buckets without explicit business justification
    4. Unencrypted data stores
    5. Missing VPC service controls
    Output a JSON with: {"findings": [...], "severity": "HIGH|MEDIUM|LOW|NONE", "blocking": true|false}
    BLOCKING means the PR must not merge without addressing this finding.""",
    tools=[github_mcp]
)

cost_agent = LlmAgent(
    name="cost_analyzer",
    model="gemini-2.5-flash",
    instruction="""Analyze the Terraform diff for GCP resource changes.
    Query BigQuery billing export to get current monthly costs for affected services.
    Estimate the monthly cost delta (increase or decrease) from this change.
    Output JSON: {"current_monthly_usd": X, "estimated_delta_usd": Y, "confidence": "HIGH|MEDIUM|LOW",
    "cost_breakdown": [{resource, current, new, delta}]}
    Flag any single-resource change adding >$500/month as HIGH_COST.""",
    tools=[bq_mcp, github_mcp]
)

compliance_agent = LlmAgent(
    name="compliance_checker",
    model="gemini-2.5-flash",
    instruction="""Check this infrastructure change against Appfire's SOC 2 Type II controls:
    1. All data stores must have encryption at rest and in transit
    2. No direct internet ingress without WAF
    3. All service accounts must have key rotation < 90 days
    4. Audit logging must be enabled on all new GCP projects
    5. No service accounts with primitive roles (Owner/Editor)
    Output JSON: {"controls_checked": [...], "violations": [...], "compliant": true|false}""",
    tools=[github_mcp]
)
```

**Step 2: Orchestrator runs them in parallel**

```python
# ParallelAgent runs all sub-agents concurrently and waits for all to complete
review_pipeline = ParallelAgent(
    name="pr_review_orchestrator",
    sub_agents=[security_agent, cost_agent, compliance_agent]
)
```

**Step 3: Post-process and comment on PR**

```python
from google.adk.agents import LlmAgent

reporter_agent = LlmAgent(
    name="report_generator",
    model="gemini-2.5-flash",
    instruction="""Merge the outputs from security_reviewer, cost_analyzer, and compliance_checker.
    Generate a structured GitHub PR comment in markdown with:
    ## AI Infrastructure Review
    ### Security [PASS/FAIL/WARNING]
    ### Cost Impact [+$X/mo | -$X/mo | Neutral]
    ### SOC 2 Compliance [PASS/FAIL]
    ### Action Required
    List any blocking items. Be direct. Engineers are busy.""",
    tools=[github_mcp]  # uses github MCP to post the comment
)
```

**Step 4: Deploy with Cloud Run webhook receiver**

```python
# cloud_run_webhook.py
import functions_framework
from google.cloud import pubsub_v1

publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path("appfire-devops", "pr-review-requests")

@functions_framework.http
def github_webhook(request):
    payload = request.get_json()
    if payload.get("action") == "opened" and "pull_request" in payload:
        publisher.publish(topic_path, data=str(payload).encode("utf-8"))
    return "OK", 200
    # Returns immediately — GitHub requires <10s response
    # Actual agent processing happens async via Pub/Sub
```

### Production Considerations

- **Parallelism wins**: Running 3 agents in parallel takes ~15–20s total.
  Sequential would take 45–60s. For PR review SLA this matters.
- **Caching**: Same PR commit SHA should not trigger re-review. Cache results
  in Firestore with SHA as key, TTL of 24h.
- **False positive rate**: Track which findings engineers override. After 4 weeks,
  retrain the agent's instruction to reduce noise on known-acceptable patterns.
- **Cost**: Each PR review costs ~$0.01–0.03 in LLM tokens + BigQuery scan.
  At 100 PRs/day, that's $30–90/month. Trivial vs. engineer time saved.

---

## Scenario 3: Cross-Platform Ops Assistant Using A2A Protocol (A2A + ADK + Agent Engine + Memory Bank)

### The Story (tell this in interviews)

> "After building the first two agents, teams across Appfire started asking if their
> tools could be part of our agent ecosystem. The ITSM team had their own ServiceNow
> agent. The CRM team had a Salesforce agent. The monitoring team had a PagerDuty
> agent. They were all built independently, on different platforms, in different
> languages. I used the A2A protocol to wire them together. Now our DevOps orchestrator
> agent — running on GCP Agent Engine — can take a natural language command like
> 'Create a P2 incident for the payments service degradation, assign it to the SRE
> on-call, pull the last 3 related incidents from ServiceNow, and post a status update
> to #incidents in Slack' and execute across 4 different agent platforms in one shot,
> with a full audit trail."

### Services Used & Why

| Service | Role | Why This One |
|---|---|---|
| **ADK (LlmAgent as orchestrator)** | Central coordinator | Interprets intent, decides which agents to call |
| **A2A Protocol** | Cross-agent communication | ServiceNow/Slack/PagerDuty agents are on external platforms |
| **Agent Card** | Agent discovery | Each external agent publishes its capabilities at `/.well-known/agent.json` |
| **Agent Engine** | Orchestrator hosting | Production-grade managed hosting for the central ADK agent |
| **Memory Bank** | Incident context | Remembers ongoing incidents, team preferences, past escalation decisions |
| **Cloud Run** | Internal A2A agents | Host lightweight GCP-native agents (GKE monitor, cost agent) |
| **Cloud Logging** | Cross-agent audit | Every A2A call logged — required for SOC 2 |
| **Secret Manager** | OAuth tokens | A2A uses OAuth 2.0 — tokens for external agents stored in Secret Manager |
| **Gemini 2.5 Pro** | Orchestrator LLM | Complex multi-step reasoning over ambiguous natural language requests |
| **GKE (Cloud Logging MCP)** | GKE log access | Reads pod logs, events via Cloud Logging MCP server |

### A2A Architecture Deep Dive

**What an Agent Card looks like (example: PagerDuty Agent)**

```json
// https://pagerduty-agent.example.com/.well-known/agent.json
{
  "name": "PagerDuty Incident Agent",
  "description": "Creates, updates, and queries PagerDuty incidents",
  "url": "https://pagerduty-agent.example.com",
  "version": "1.2.0",
  "skills": [
    {
      "id": "create_incident",
      "name": "Create Incident",
      "description": "Create a new PagerDuty incident",
      "inputModes": ["text/plain", "application/json"],
      "outputModes": ["application/json"]
    },
    {
      "id": "get_oncall",
      "name": "Get On-Call",
      "description": "Return the current on-call engineer for a given service",
      "inputModes": ["text/plain"],
      "outputModes": ["application/json"]
    }
  ],
  "authentication": {
    "schemes": ["oauth2"],
    "oauth2": {
      "flows": {
        "clientCredentials": {
          "tokenUrl": "https://pagerduty-agent.example.com/oauth/token",
          "scopes": {"incident:write": "Create incidents", "oncall:read": "Read on-call schedule"}
        }
      }
    }
  }
}
```

**Why this matters:** The orchestrator agent reads this JSON to discover what the
PagerDuty agent can do — without you writing any custom integration code. If the
PagerDuty team adds a new skill, it appears automatically on next discovery.

### Step-by-Step Implementation

**Step 1: Build A2A-aware tool wrappers in ADK**

```python
import httpx
import json
from google.adk.tools import FunctionTool
from google.cloud import secretmanager

def get_oauth_token(agent_url: str) -> str:
    """Fetch OAuth token for an A2A agent from Secret Manager."""
    client = secretmanager.SecretManagerServiceClient()
    secret_name = f"a2a-token-{agent_url.replace('https://', '').replace('.', '-')}"
    response = client.access_secret_version(
        request={"name": f"projects/appfire-devops/secrets/{secret_name}/versions/latest"}
    )
    return response.payload.data.decode("utf-8")

def call_a2a_agent(agent_url: str, skill_id: str, payload: dict) -> dict:
    """
    Send a task to an A2A-compatible agent and poll for result.
    This is the A2A client implementation pattern.
    """
    token = get_oauth_token(agent_url)
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }

    # A2A task submission (JSON-RPC over HTTP)
    task_request = {
        "jsonrpc": "2.0",
        "method": "tasks/send",
        "id": "1",
        "params": {
            "skill": skill_id,
            "message": {
                "role": "user",
                "parts": [{"type": "text", "text": json.dumps(payload)}]
            }
        }
    }

    response = httpx.post(f"{agent_url}/", json=task_request, headers=headers)
    result = response.json()

    # Poll for completion (or use SSE for streaming)
    task_id = result["result"]["id"]
    while True:
        poll = httpx.get(
            f"{agent_url}/tasks/{task_id}",
            headers=headers
        )
        task = poll.json()["result"]
        if task["status"]["state"] in ["completed", "failed"]:
            return task
        import time; time.sleep(1)


# Define as ADK tools
pagerduty_tool = FunctionTool(
    func=lambda service, severity, summary: call_a2a_agent(
        "https://pagerduty-agent.example.com",
        "create_incident",
        {"service": service, "severity": severity, "summary": summary}
    ),
    name="create_pagerduty_incident",
    description="Create a PagerDuty incident for a production service issue"
)

servicenow_tool = FunctionTool(
    func=lambda query: call_a2a_agent(
        "https://servicenow-agent.internal",
        "search_incidents",
        {"query": query}
    ),
    name="search_past_incidents",
    description="Search ServiceNow for past incidents related to a service or error type"
)
```

**Step 2: Build the orchestrator agent**

```python
from google.adk.agents import LlmAgent
from google.adk.memory import VertexAiMemoryBankService
from google.adk.tools.mcp import MCPToolset, SseServerParams

# Cloud Logging MCP for GKE log access
logging_mcp = MCPToolset(
    connection_params=SseServerParams(
        url="https://mcp-server-cloudlogging.run.app/sse"
    )
)

memory_service = VertexAiMemoryBankService(
    project="appfire-devops",
    location="us-central1"
)

ops_orchestrator = LlmAgent(
    name="devops_ops_assistant",
    model="gemini-2.5-pro",
    instruction="""You are the DevOps Operations AI for Appfire.
    You coordinate across multiple systems to handle production incidents and operational tasks.

    ALWAYS follow this incident response order:
    1. Check Cloud Logging for error patterns in the affected service
    2. Search ServiceNow for past similar incidents and their resolutions
    3. Create a PagerDuty incident if severity is P1 or P2
    4. Notify the Slack #incidents channel with a structured update

    When you have context from Memory Bank about an ongoing incident, use it.
    Do not re-ask the engineer for information they already gave you this session.

    Be concise. Engineers in incidents need facts, not explanations.""",
    tools=[
        logging_mcp,
        pagerduty_tool,
        servicenow_tool,
        slack_notify_tool,  # similar A2A wrapper for Slack agent
    ],
    memory_service=memory_service
)
```

**Step 3: Deploy orchestrator to Agent Engine**

```python
import vertexai
from vertexai.preview import agent_engines

vertexai.init(project="appfire-devops", location="us-central1")

# Package the agent with all dependencies
remote_agent = agent_engines.create(
    ops_orchestrator,
    display_name="devops-ops-assistant-v1",
    requirements=[
        "google-cloud-aiplatform[adk,agent_engines]",
        "httpx",
        "google-cloud-secret-manager"
    ]
)

print(f"Agent deployed at: {remote_agent.resource_name}")
```

**Step 4: Wire up to Slack for natural language input**

```python
# slack_bot.py — receives Slack slash command /ops [query]
@app.route("/slack/ops", methods=["POST"])
def handle_ops_command():
    user_query = request.form.get("text")
    user_id = request.form.get("user_id")

    session = remote_agent.chat(
        user_id=user_id,
        message=user_query
    )

    return jsonify({
        "response_type": "in_channel",
        "text": session.response
    })
```

### Real Example Interactions (demonstrate in interview)

```
Engineer: /ops payments service is returning 503s on checkout, looks like it started 20 mins ago

Agent:
→ [Queries Cloud Logging] Found 847 errors in payments-service pod in us-central1-a since 14:23 UTC.
  Root cause pattern: connection pool exhausted to Cloud SQL (max_connections: 100, current: 102)

→ [Searches ServiceNow] Found 2 past incidents:
  INC-4521 (Mar 2025): Same root cause, resolved by increasing max_connections to 200 and restarting
  INC-3890 (Nov 2024): Similar but caused by a slow query — different fix

→ [Creates PagerDuty] Created INC-P1-8842, assigned to Sarah Chen (current on-call for payments)

→ [Notifies Slack #incidents] Posted: "P1 ACTIVE — Payments 503s — Connection pool exhausted.
  Sarah Chen paged. Likely fix: increase max_connections. Last resolved in 8 min."

Summary: Connection pool issue. Past resolution took 8 minutes. Sarah has been paged.
Runbook: https://confluence.appfire.com/runbooks/cloudsql-connection-pool
```

### Production Considerations

- **A2A timeout handling**: External agents can be slow. Set a 30s timeout on A2A calls.
  If an agent times out, the orchestrator should proceed without it and note the gap.
- **Circuit breaker**: If PagerDuty agent fails 3 times in 5 minutes, fall back to
  direct PagerDuty API call. Don't let one failed external agent block an incident response.
- **A2A authentication rotation**: OAuth tokens expire. Implement auto-refresh in the
  `get_oauth_token` function. Store refresh tokens in Secret Manager, not environment variables.
- **Audit logging for A2A**: Every A2A task sent and received gets logged to Cloud Logging
  with the session ID, agent name, skill called, and response status. This is your audit
  trail for SOC 2 auditors asking "what did your AI agent do last Tuesday at 3 AM?"

---

## Part 3: Interview Questions & Answers

### Section A: Foundational GenAI & RAG

**Q1. What is RAG and why do you use it instead of fine-tuning?**

RAG retrieves relevant documents at inference time and injects them into the prompt.
Fine-tuning bakes knowledge into the model weights. For operational data (runbooks,
incident reports, pricing data) that changes weekly, RAG is the right choice — you update
the corpus without retraining the model. Fine-tuning is for teaching the model a style,
reasoning pattern, or domain vocabulary that doesn't change. In production, almost always
start with RAG; only fine-tune when RAG's retrieval quality ceiling has been proven
insufficient.

**Q2. How do you evaluate a RAG pipeline?**

Three metrics: retrieval precision (are the top-k chunks actually relevant?), answer
faithfulness (does the response only use what was retrieved, or is the model hallucinating
additions?), and answer relevance (does the response actually answer the question?).
We use a golden eval set: 50 curated Q&A pairs where we know the correct answer and source
document. Run this weekly. Track all three metrics in a dashboard. If retrieval precision
drops, check if new documents broke chunk boundaries. If faithfulness drops, tighten the
system prompt instruction to cite sources.

**Q3. What is chunk_overlap in RAG and why does it matter?**

When a document is split into chunks, overlap means the last N tokens of chunk 1 are
repeated at the start of chunk 2. Without overlap, a sentence that spans a chunk boundary
is split, losing its meaning in both chunks. With overlap, both chunks carry enough context
to be retrievable. Too much overlap wastes embedding space and retrieval capacity. Standard
is 10–20% of chunk_size. For runbooks where steps often span paragraph boundaries, I use
100-token overlap on 512-token chunks.

**Q4. What is grounding and how does it differ from RAG?**

Grounding is the higher-level concept — ensuring the LLM response is anchored to a
verifiable source. RAG is one implementation of grounding (your private docs). Google
Search Grounding is another (live public web). In Vertex AI, you can stack them:
ground on your internal corpus first, fall back to Google Search for public facts.
Grounding gives you citations; citations give you auditability; auditability is required
for any enterprise or regulated deployment.

---

### Section B: ADK & Agent Architecture

**Q5. What are the main agent types in ADK and when do you choose each?**

- `LlmAgent`: Any task requiring LLM reasoning, tool use, or natural language understanding.
  The default choice.
- `SequentialAgent`: When sub-agents must run in order and each needs the previous agent's
  output. Example: ingest → analyse → report pipeline.
- `ParallelAgent`: When sub-agents are independent and you want to minimise wall-clock time.
  Example: security + cost + compliance review running concurrently.
- `LoopAgent`: When you need to retry until a condition is met. Example: polling an
  external API until a deployment reaches a `SUCCEEDED` status.

**Q6. How do you handle agent state across a multi-turn conversation?**

ADK's session service maintains the conversation history. For cross-session state (things
the agent should remember next week), I use Memory Bank. The pattern: after each session
ends, extract key facts (current incident ID, affected service, decisions made) and store
them as memories. On the next session, inject the relevant memories into the system prompt
context. This gives the agent continuity without blowing up the context window with full
history.

**Q7. How do you ensure an agent doesn't hallucinate tool calls?**

Three controls: First, tool descriptions must be precise — vague tool descriptions cause
the agent to guess when to call them. Second, use `temperature: 0.1` or lower for agents
that call real-world tools — lower temperature makes tool selection more deterministic.
Third, validate tool inputs before execution. Wrap each tool call in a Pydantic model that
validates the LLM-provided arguments before the actual API call runs.

**Q8. How do you test ADK agents?**

Unit test each tool function independently — the LLM is not involved. Integration test
each agent with a fixed set of input messages and assert on tool call patterns (which tool
was called, with what arguments). End-to-end test the full pipeline on a staging
environment with synthetic but realistic data. Track eval metrics over time. Agents degrade
silently when the underlying model changes — continuous evals are the only way to catch it.

---

### Section C: Multi-Agent & A2A

**Q9. What is the A2A protocol and why does it exist?**

A2A is an open HTTP standard for agent-to-agent communication across platforms. It exists
because the enterprise has heterogeneous agent ecosystems — a GCP agent needs to call a
ServiceNow agent, a Salesforce agent, a custom internal agent — and without a standard
protocol, every pair of agents needs a custom integration. A2A standardises discovery
(Agent Card), task submission (JSON-RPC), transport (HTTP + SSE), and authentication
(OAuth 2.0 + JWT). As of 2026, it is governed by the Linux Foundation and in production
at 150+ enterprises.

**Q10. What is an Agent Card and what does it contain?**

An Agent Card is a JSON document at `/.well-known/agent.json` on any A2A-compliant agent
server. It declares: the agent's name, URL, version, skills (what tasks it can perform),
input/output modes (text, JSON, images), and authentication scheme (OAuth 2.0 scopes).
It is the agent equivalent of an OpenAPI spec. Orchestrators read Agent Cards to discover
what agents exist and what they can do. If an agent adds a new skill, the Card updates
automatically — no code changes needed in the orchestrator.

**Q11. How does authentication work in A2A?**

A2A uses OAuth 2.0 client credentials flow. The orchestrator agent has a service account
identity. When it calls an external agent, it presents a JWT signed by its GCP service
account key (stored in Secret Manager). The external agent validates the JWT against the
expected issuer and audience, then returns an access token scoped to the specific skills
requested. No credentials are ever passed in the task payload — only in the Authorization
header. Token refresh is handled automatically; tokens are cached in memory and refreshed
when they approach expiry.

**Q12. How do you handle failures in a multi-agent pipeline?**

Three patterns: First, each agent in a ParallelAgent should fail independently — a failed
cost agent should not block the security agent result. Use try/except in tool implementations
and return a structured error response that the orchestrator can include in the report.
Second, circuit breaker: if an external A2A agent fails 3+ times in a window, stop calling
it and raise an alert instead of hammering a failing service. Third, idempotency: agent
tasks that create external resources (PagerDuty incidents, Jira tickets) must check if the
resource already exists before creating it to avoid duplicates on retry.

---

### Section D: Production, MLOps & Observability

**Q13. How do you monitor an agentic AI system in production?**

Three layers: correctness, performance, cost. For correctness: run continuous evals against
a golden dataset; alert if accuracy drops >5% week-over-week. For performance: trace every
agent turn with Cloud Trace — track p50/p95/p99 latency for tool calls and LLM calls
separately. For cost: track tokens per request in Cloud Monitoring, set budget alerts in
Cloud Billing. The most important metric no one tracks: "agent turn completion rate" —
how often does the agent finish its task in one turn vs. requiring follow-up clarification?
Low completion rate means the agent is misunderstanding requests.

**Q14. What is prompt versioning and why does it matter for production agents?**

Your agent's system prompt is code. It must be version-controlled (Git), tested before
deployment, and rolled back if it degrades performance. Store prompts in a config file,
not hardcoded strings. When you change the prompt, run the full eval suite before deploying.
Tag every LLM call in Cloud Logging with the prompt version so you can correlate metric
changes to prompt changes. A prompt change that silently broke your agent two weeks ago
is very hard to debug without this.

**Q15. How do you handle model updates (e.g., Gemini 2.5 → Gemini 3.0)?**

Run the new model against your golden eval set before switching. Compare accuracy, latency,
and cost per token. If new model shows regression on any golden test case, file it as a
known issue and decide whether to delay upgrade or update the eval expectation. Never
switch model versions in production without a regression test. In GCP, you can pin to
a specific model version (e.g., `gemini-2.5-pro-001`) to avoid automatic upgrades.

**Q16. How do you manage cost for a production agentic AI system?**

Instrument every LLM call with input token count, output token count, and model name.
Export to BigQuery via Cloud Logging. Build a Looker Studio dashboard showing cost per
agent, cost per tool call, and cost trend over time. Set budget alerts at 80% of monthly
cap. Optimise: use Flash models for structured tasks (cost analysis, compliance checking),
reserve Pro models for open-ended reasoning. Cache repeated queries — if 30% of your
queries ask the same thing (e.g., "what's the on-call rotation?"), cache the answer for
15 minutes and save 30% on token costs.

**Q17. How do you implement SOC 2 compliance for an AI agent?**

Four controls: First, data retention — log every agent input and output for 90 days with
the user identity who triggered it. Second, access control — agent personas are GCP service
accounts with IAM roles scoped to exactly the data they need; no service account has
Editor or Owner. Third, audit trail — every A2A call, every tool call, every LLM call is
logged with timestamp, actor, and result to Cloud Logging with log sinks to a tamper-
evident bucket. Fourth, data residency — configure Vertex AI to process data in a specific
region; for EU customers, process in `europe-west1` only.

---

### Section E: Architecture Decisions

**Q18. When do you use RAG Engine vs building a custom RAG pipeline?**

RAG Engine for 80% of use cases — it handles chunking, embedding, storage, and retrieval
with a managed API. Build custom when: you need a non-standard embedding model (fine-tuned
on domain vocab), you need hybrid search (BM25 + semantic), or you need to integrate an
existing Elasticsearch or Pinecone index you cannot migrate. Custom RAG is 3–5x more
engineering effort; only worth it when RAG Engine's ceiling is genuinely blocking you.

**Q19. When do you deploy agents on Agent Engine vs GKE?**

Agent Engine for most production use cases: managed, auto-scaling, integrated observability.
GKE when: you need GPU-attached nodes for running your own model inference alongside the
agent, you need custom networking (private VPC-only agent with no public internet), you
have complex sidecar dependencies (Istio service mesh, custom cert management), or you need
to co-locate agents with other Kubernetes workloads to reduce latency.

**Q20. How do you design a multi-agent system that is maintainable over 12 months?**

Three principles: First, single-responsibility — each agent does one thing well. A cost
agent that also does security review is a maintenance nightmare. Second, contract-first —
define each agent's input/output schema as a Pydantic model before writing the agent.
When the agent changes internally, the contract protects callers. Third, observability
from day one — if you cannot trace which agent made which decision and why, you cannot
debug production issues. Bolt-on observability is 5x harder than building it in. Use
Cloud Trace spans and structured JSON logging on every agent from the first deployment.

---

### Section F: Quick-Fire Scenario Questions

**Q21. Your RAG agent's answers degrade over 2 weeks. What do you investigate first?**

Check corpus freshness first — were new documents added that are breaking retrieval?
Next, check if the underlying embedding model was silently updated. Then run the golden
eval set to identify which question categories degraded. If retrieval precision dropped,
check chunk boundaries and document structure. If faithfulness dropped, the model may
be ignoring retrieved context — strengthen the system prompt instruction to use only
provided sources.

**Q22. An A2A agent you depend on is returning 50% errors. What do you do?**

Check Agent Card first — has the skill ID or input schema changed? If yes, update your
tool wrapper. If the Agent Card is unchanged, check the error responses: are they
authentication failures (token rotation needed?), rate limits (add backoff), or 5xx
errors (their problem, activate your circuit breaker). Log all A2A errors with the
full request payload so you can give their team a reproduction case. Fall back to
direct API calls if the agent is unavailable for >5 minutes.

**Q23. You need to add a new external agent to your A2A ecosystem. What is the process?**

Fetch and review their Agent Card. Confirm authentication scheme matches what you support.
Write a tool wrapper using the A2A client pattern. Add the OAuth credentials to Secret
Manager. Write unit tests for the tool wrapper with mocked A2A responses. Write
integration tests against their staging endpoint. Add the tool to the relevant orchestrator
agent and update its system prompt to describe when to use the new tool. Deploy to staging
and run end-to-end tests before production.

**Q24. How would you explain the difference between MCP and A2A to a non-technical stakeholder?**

MCP (Model Context Protocol) is about connecting AI agents to data sources and tools —
like giving an agent access to your database, files, or APIs. Think of it as the agent's
hands. A2A (Agent2Agent) is about AI agents talking to other AI agents — like agents
delegating work to specialist colleagues on other teams. MCP is vertical integration
(agent to tool). A2A is horizontal integration (agent to agent).

**Q25. What would you build first if a company asked you to "add AI to our DevOps workflow"?**

Start with the highest-friction, lowest-risk task: not autonomous deployments, but
information retrieval. Build a RAG agent on top of your existing runbooks and post-mortems.
It removes friction for on-call engineers without any risk of autonomous action. Prove
value fast (measurable: mean time to acknowledge drops). Then layer in agentic automation
incrementally: start with read-only tool calls (log queries, metric reads), then write
operations with human approval gates, then supervised autonomy. Never start with write
autonomy — you earn that trust from the team over months, not weeks.

---

## Quick Reference: Service Selection Cheat Sheet

```
I need to...                           Use...
─────────────────────────────────────────────────────────────────────
Ground LLM in my private docs         RAG Engine + Grounding
Search millions of vectors fast       Vertex AI Vector Search (bring your own)
Build a custom agent with logic       ADK (LlmAgent, SequentialAgent, etc.)
Run agents in parallel                ADK ParallelAgent
Keep agent memory across sessions     Memory Bank
Host an agent (managed)               Agent Engine (Deployments)
Host an agent (custom infra/GPU)      GKE + ADK
Connect agents across platforms       A2A Protocol
Expose my API as an agent tool        MCP Server (Cloud Run)
Query BigQuery from an agent          BigQuery MCP Server
Read Cloud Logs from an agent         Cloud Logging MCP Server
Build stateful, cyclic workflows      LangGraph (within ADK)
Publish agent capabilities            Agent Card (/.well-known/agent.json)
Store secrets for A2A OAuth           Secret Manager
Trace multi-agent execution           Cloud Trace
Monitor token costs per agent         Cloud Logging → BigQuery → Looker Studio
```

---

*Sources:*
- [Gemini Enterprise Agent Platform — Google Cloud](https://cloud.google.com/products/agent-builder)
- [ADK with A2A Protocol — Google ADK Docs](https://google.github.io/adk-docs/a2a/)
- [RAG Engine Overview — Google Cloud Docs](https://docs.cloud.google.com/gemini-enterprise-agent-platform/build/rag-engine/rag-overview)
- [Build Multi-Agent Systems with ADK — Google Codelabs](https://codelabs.developers.google.com/codelabs/production-ready-ai-with-gc/3-developing-agents/build-a-multi-agent-system-with-adk)
- [Deploy ADK on GKE with Vertex AI — Google Cloud Docs](https://docs.cloud.google.com/kubernetes-engine/docs/tutorials/agentic-adk-vertex)
- [Build Cross-Language Multi-Agent Team with ADK and A2A — Google Developers Blog](https://developers.googleblog.com/build-cross-language-multi-agent-team-with-google-agent-development-kit-and-a2a/)
- [RAG and Grounding on Vertex AI — Google Cloud Blog](https://cloud.google.com/blog/products/ai-machine-learning/rag-and-grounding-on-vertex-ai)
- [Production-Grade RAG with ADK & Vertex AI — Google Cloud Community](https://medium.com/google-cloud/how-to-build-a-production-grade-rag-with-adk-vertex-ai-rag-engine-via-the-agent-starter-pack-7e39e9cfe856)
