# GCP Agentic AI — Interview Mastery Guide

> Written for Vinay, data / cloud engineer at Wipro working on the Fabric data client
> project, preparing for senior agentic AI interviews on GCP.
> Every scenario reflects real daily work. Every service definition explains why you
> chose it. Every step tells you exactly what to click or run. Use these as talking
> points, not scripts.

---

## Part 1: GCP Agentic AI Services — Definitions & Why You Use Each

Understanding *why* you chose each service is what separates someone who read the docs
from someone who actually built it in production.

---

### 1. Gemini Enterprise Agent Platform (formerly Vertex AI Agent Builder)

**What it is:**
Google's unified platform for building, deploying, and governing AI agents in production.
Rebranded at Cloud Next 2026. Bundles ADK, Agent Studio (low-code visual builder), Agent
Engine (managed runtime), RAG Engine, Memory Bank, and 200+ models under one roof.

**Why you use it:**
You don't want to stitch together separate ML services, vector databases, and hosting
infrastructure yourself. This gives you the full stack — model serving, retrieval, agent
runtime, observability — on a single managed platform with IAM baked in from the start.

**When to use vs alternatives:**
Use this when your primary backbone is GCP. If you need multi-cloud or custom networking,
you deploy ADK on GKE instead and manage your own runtime.

---

### 2. Agent Development Kit (ADK)

**What it is:**
Google's open-source framework (Python, Go, Java, TypeScript) for building agents in code.
Released GA at Cloud Next 2026. The same framework that powers Google Agentspace and
Customer Engagement Suite. Supports sequential, parallel, hierarchical, and loop-based
multi-agent patterns.

**Why you use it:**
ADK gives you production-grade control that the low-code Agent Studio cannot match. You
define exactly which tools the agent can call, manage state explicitly, write custom routing
logic, and plug in any model — Gemini, Claude, or open-source. For enterprise use cases
with complex branching logic, ADK is the right choice on GCP.

**Key agent types:**
- LlmAgent — base agent backed by an LLM, handles reasoning and tool use
- SequentialAgent — runs sub-agents one after another, passes output as context
- ParallelAgent — runs sub-agents at the same time, merges all results at the end
- LoopAgent — keeps running until a termination condition is met (useful for polling)

**Deployment options:** Agent Engine (fully managed, no infra to manage) or GKE (self-managed, full control).

---

### 3. Agent Engine (also called "Deployments")

**What it is:**
The managed serverless runtime for deploying ADK agents. Handles auto-scaling, session
management, health checks, and observability without you managing any Kubernetes.
Supports sub-second cold starts and long-running workflows.

**Why you use it:**
Agent workloads are bursty — zero requests at night, spike during business hours or
incidents. Agent Engine auto-scales to zero and back without you touching a single pod.
You deploy your packaged ADK agent and get back a REST endpoint immediately. Integrates
with Cloud Monitoring, Cloud Logging, and Cloud Trace out of the box.

**Use case:** Any agent serving production traffic with variable load and session state requirements.

---

### 4. A2A — Agent2Agent Protocol

**What it is:**
An open HTTP-based protocol (governed by the Linux Foundation as of 2026) that lets AI
agents on different platforms talk to each other — without sharing internal code,
credentials, or architecture. 150+ organisations use it in production including Salesforce,
ServiceNow, SAP, Accenture, and Deloitte.

**Core concepts:**
- **Agent Card** — a JSON file at `/.well-known/agent.json` on every A2A agent server.
  It lists the agent's name, URL, version, skills, what inputs it accepts, what outputs
  it produces, and how to authenticate. It is the agent equivalent of an OpenAPI spec.
- **Task** — the unit of work. An orchestrator sends a task to a specialist agent and
  either polls for the result or streams it via Server-Sent Events.
- **Transport** — built on standard HTTP, JSON-RPC, and Server-Sent Events. No
  proprietary binary protocol. Any language or platform can implement it.
- **Auth** — OAuth 2.0 + JWT. Agents authenticate each other without passing secrets
  in the request body.

**Why you use it:**
In enterprise you never control every system. Your GCP agent needs to call a ServiceNow
agent the ITSM team manages, a Collibra agent the data governance team owns, a PagerDuty
agent the SRE team runs. A2A lets you do that without writing a custom integration for
every pair of agents. One standard protocol, all platforms.

---

### 5. RAG Engine

**What it is:**
A fully managed Retrieval-Augmented Generation (RAG) service on the Gemini Enterprise
Agent Platform. Manages the full pipeline: document ingestion from GCS or Google Drive,
chunking, embedding generation, vector storage (default: Spanner-backed managed store),
retrieval, and context injection into the LLM prompt before it generates a response.

**Why you use it:**
RAG solves hallucination at the data layer. Without it, the LLM makes up answers from
its training data. With it, every response is grounded in your actual documents —
runbooks, data dictionaries, post-mortems, architecture docs. RAG Engine handles the
messy parts (keeping embeddings fresh, tuning chunk size, hybrid search) so you focus
on the application, not the infrastructure.

**RAG vs fine-tuning:**
RAG is right when your data changes frequently — runbooks, pipeline schemas, pricing.
Fine-tuning is for baking in a fixed reasoning style or domain vocabulary. In data
platform operations, almost always start with RAG. Fine-tune only after proving RAG's
retrieval ceiling is the bottleneck.

---

### 6. Vertex AI Vector Search

**What it is:**
Google's scalable, low-latency approximate nearest neighbour (ANN) search service.
Used for similarity search across millions of vector embeddings. Built on the same
technology behind Google Search's internal embedding retrieval.

**Why you use it over RAG Engine's built-in store:**
When you already have embeddings generated by your own fine-tuned encoder model, or
when you need to query tens of millions of vectors with sub-100ms latency SLA, you
bring your own index in Vector Search instead of RAG Engine's managed store. Also
the right choice for multi-modal search (images + text together).

---

### 7. Grounding

**What it is:**
Grounding ensures a Gemini response is anchored to a verifiable source before being
returned to the user. Two types on GCP:
- **Google Search Grounding** — Gemini queries live public Google Search to verify facts.
- **RAG Grounding** — Gemini queries your private vector store via RAG Engine.

**Why you use it:**
Without grounding, LLMs are confident liars. With grounding, every claim in the response
can be traced back to a specific document and section. In regulated environments —
finance, healthcare, SOC 2-audited orgs — you cannot deploy LLMs without grounding.
An auditor will ask where the response came from. You need an answer.

---

### 8. Memory Bank

**What it is:**
A managed persistent memory store for agents. Stores long-term context across sessions
so your agent remembers what it learned in last week's conversation — the team's infra
layout, current incident context, user preferences — and does not ask again next week.

**Why you use it:**
Stateless agents are frustrating in production. If your data ops assistant forgets
that the Fabric Lakehouse cluster was resized two days ago, it gives outdated answers.
Memory Bank gives agents continuity across sessions without you building a custom
state management database. It stores both exact key-value facts and semantic memories
that can be retrieved by meaning, not just exact match.

**Use case:** Storing ongoing incident context, team configuration decisions, pipeline
topology, and user preferences that improve agent accuracy over time.

---

### 9. MCP — Model Context Protocol

**What it is:**
An open standard (created by Anthropic, adopted by Google and all major AI platforms)
for exposing tools and data sources to LLMs in a standardised way. An MCP server
exposes a set of callable tools over a well-known HTTP interface. ADK supports MCP
natively — you point your agent at an MCP server URL and it discovers all available
tools automatically.

**Pre-built GCP MCP servers:** BigQuery, Google Maps, Cloud Logging, Gmail, Google Drive,
Cloud Storage.

**Why you use it:**
Instead of writing custom API wrappers for every data source your agent needs to reach,
you run an MCP server. Your BigQuery MCP server means any agent in your ecosystem can
query your billing or pipeline data with a single tool call — no hardcoded queries,
no custom integration per agent.

---

### 10. LangGraph

**What it is:**
A Python framework from LangChain for building stateful, cyclic agent workflows as
explicit state machines. Lets you define nodes (agent steps), edges (transitions), and
conditional routing between them. ADK integrates LangGraph agents as sub-agents.

**Why you use it:**
When a workflow is not a simple sequence — when an agent needs to loop, retry, branch
on intermediate results, or maintain complex state across 15 tool calls — LangGraph's
explicit state machine is far safer than a free-form ReAct loop. You can draw the
workflow, test individual nodes in isolation, and trace exact execution paths in Cloud Trace.

---

### 11. Cloud Run (for A2A agent hosting)

**What it is:**
Google's serverless container platform. Pay only when requests come in, scales to zero,
deploys any containerised application. Ideal for hosting lightweight A2A-compliant
specialist agents.

**Why you use it:**
A specialist "skill agent" in A2A terms gets called occasionally by an orchestrator.
Cloud Run costs nothing at idle and handles bursts without you managing instances.
An Agent Card is a static JSON file — serve it from Cloud Run alongside the agent itself.

---

### 12. GKE — Google Kubernetes Engine (for ADK at scale)

**What it is:**
Google's managed Kubernetes service. When Agent Engine's managed runtime is too
constraining — custom networking, GPU-attached nodes, non-standard sidecar dependencies
— you deploy ADK agents on GKE yourself and manage the runtime.

**Why you use it over Agent Engine:**
Large multi-agent systems needing GPU inference for local model serving, strict VPC-only
networking with no public endpoints, or complex Istio service mesh integration need GKE.
Agent Engine is the right default — GKE is for when you have genuinely outgrown it.

---

### 13. Cloud Logging + Cloud Trace + Cloud Monitoring (Observability Stack)

**Why this matters specifically for agentic AI:**
An agent that makes 12 tool calls before answering is a complete black box without
tracing. Cloud Trace shows you the full execution span of every agent turn — which tool
was called, what arguments it received, what it returned, and how long it took. Without
this, debugging a misbehaving agent is guesswork.

**What you must instrument on every agent:**
- Every tool call: input arguments, output result, latency
- Every LLM call: model name, input tokens, output tokens, cost estimate
- Every Memory Bank read and write
- Every A2A outbound and inbound call with latency

---

### 14. Cloud Pub/Sub (for async agent triggering)

**What it is:**
Google's managed message queue. Decouples event producers (webhooks, pipeline triggers)
from consumers (agents). Guarantees at-least-once delivery with configurable retry.

**Why you use it:**
GitHub webhooks and data pipeline event notifications have a hard 10-second response
timeout. Your agent takes 30–90 seconds to run. Without Pub/Sub, the webhook times out
and GitHub marks the delivery as failed. With Pub/Sub, your webhook receiver publishes
the message instantly and returns 200, then the agent processes it asynchronously from
the queue at its own pace.

---

### 15. Secret Manager

**What it is:**
Google's managed secrets store. Stores API keys, OAuth tokens, service account
credentials, and any sensitive configuration. Integrates with IAM — only identities
you explicitly grant can access a secret.

**Why you use it:**
A2A agents authenticate via OAuth 2.0 tokens. Those tokens must never be hardcoded in
agent config or environment variables. Secret Manager gives you centralised, auditable,
rotation-capable credential storage. Every secret access is logged, which is a SOC 2
requirement.

---

## Part 2: The 3 Production Scenarios

---

## Scenario 1: Data Platform Incident Intelligence System
### (RAG Engine + Grounding + ADK + Memory Bank + Agent Engine)

### The Story (tell this in interviews)

> "At Wipro, on the Fabric data client project, our data platform team was drowning
> in alert noise. We had 500+ runbooks, post-mortems, and architecture docs spread
> across Confluence, Google Drive, and GitHub. When a P1 data pipeline failure came
> in at 3 AM, the on-call engineer spent 20 minutes searching docs before even starting
> to fix it. I built an Incident Intelligence agent that lets on-call engineers ask in
> plain English: 'The Fabric Lakehouse ingestion pipeline is failing on the sales dataset,
> what do I do?' — and it returns the relevant runbook steps, the last 3 similar
> incidents, and the escalation path in under 5 seconds. We reduced mean time to
> acknowledge from 22 minutes to under 4 minutes."

---

### Services Used & Why

| Service | Role | Why This One |
|---|---|---|
| **Gemini 2.5 Pro** | LLM backbone | Best long-context reasoning over dense technical docs |
| **RAG Engine** | Private knowledge retrieval | Fully managed — no custom vector DB infra needed |
| **Cloud Storage (GCS)** | Document source | All runbooks and post-mortems stored as Markdown/PDF |
| **ADK** | Agent orchestration | Custom pre/post processing logic around the RAG retrieval |
| **Grounding** | Source citation | Engineers must see exactly which doc the answer came from |
| **Agent Engine** | Serving | Managed runtime, auto-scales for bursty incident traffic |
| **Memory Bank** | Session continuity | Agent remembers incident context across follow-up questions |
| **Cloud Logging** | Audit trail | SOC 2 requirement — log every query, response, and source doc |
| **Secret Manager** | API credentials | Slack bot token and service account keys stored securely |

---

### Step-by-Step: Building the Incident Intelligence Agent

---

#### Step 1 — Enable Required APIs in GCP Console

1. Go to GCP Console → **APIs & Services** → **Library**
2. Search and enable each of the following one by one:
   - **Vertex AI API**
   - **Cloud Storage API**
   - **Cloud Logging API**
   - **Secret Manager API**
   - **Cloud Run API**
3. Wait for each API to show status **Enabled** before proceeding.

**Why:** GCP services are disabled by default per project. Without enabling these APIs,
every call returns a permission error that looks like a credentials problem.

---

#### Step 2 — Create a GCP Project and Set Up IAM

1. Go to GCP Console → top bar → **Select a project** → **New Project**
2. Name it `wipro-fabric-data`, select your billing account, click **Create**
3. Go to **IAM & Admin** → **Service Accounts** → **Create Service Account**
4. Name: `fabric-agent-sa`, click **Create and Continue**
5. Assign these roles:
   - `Vertex AI User` — allows calling Gemini and RAG Engine
   - `Storage Object Viewer` — allows reading runbooks from GCS
   - `Logging Log Writer` — allows writing agent logs to Cloud Logging
   - `Secret Manager Secret Accessor` — allows reading OAuth tokens from Secret Manager
6. Click **Done**
7. Click the service account name → **Keys** tab → **Add Key** → **Create new key** → JSON
8. Download the JSON key file. Store it securely — this is used by your ADK agent.

**Why least-privilege roles:** If this service account is ever compromised, an attacker
can only read storage and call Vertex AI — not modify infrastructure or access other projects.
This is a SOC 2 access control requirement.

---

#### Step 3 — Upload Runbooks and Post-Mortems to Cloud Storage

1. Go to GCP Console → **Cloud Storage** → **Buckets** → **Create**
2. Name: `wipro-fabric-runbooks`
3. Region: `us-central1` (keep all resources in same region to avoid cross-region egress costs)
4. Storage class: **Standard**
5. Access control: **Uniform** (bucket-level IAM only — no per-object ACLs)
6. Click **Create**
7. Open the bucket → **Upload files**
8. Upload all runbook files (PDF, Markdown, DOCX all supported)
9. Repeat: create a second bucket named `wipro-fabric-postmortems` and upload post-mortem docs
10. Go to **Permissions** tab on each bucket → **Grant Access**
11. Add principal: `fabric-agent-sa@wipro-fabric-data.iam.gserviceaccount.com`
12. Role: **Storage Object Viewer** → **Save**

**Why separate buckets:** Runbooks and post-mortems have different update frequencies and
access patterns. Separating them makes it easy to refresh only post-mortems without
re-ingesting runbooks, saving embedding cost.

---

#### Step 4 — Create a RAG Corpus in Vertex AI

1. Go to GCP Console → **Vertex AI** → **Agent Builder** (or search "RAG Engine")
2. Click **RAG Engine** in the left sidebar
3. Click **Create Corpus**
4. Fill in:
   - Display name: `fabric-runbooks-corpus`
   - Description: `All runbooks, post-mortems, and data platform architecture docs for Wipro Fabric client`
   - Embedding model: **text-embedding-005** (latest stable, best retrieval quality)
5. Click **Create**
6. Once corpus is created, click **Import Files**
7. Source: **Google Cloud Storage**
8. GCS path: `gs://wipro-fabric-runbooks/` → click **Add another path** → `gs://wipro-fabric-postmortems/`
9. Chunking settings:
   - Chunk size: **512 tokens** — optimal for structured procedural runbooks
   - Chunk overlap: **100 tokens** — prevents losing context at chunk boundaries
10. Click **Import** and wait for the status to show **Completed**

**Why chunk size 512:** Runbooks have dense step-by-step content. Chunks smaller than
256 tokens lose the context between adjacent steps. Chunks larger than 1024 dilute
retrieval precision because one chunk covers too many different topics. 512 is the
empirically validated sweet spot for operational documentation.

---

#### Step 5 — Test Retrieval in the Console

1. In RAG Engine → click your corpus → **Test Retrieval** tab
2. Type a query: `Fabric Lakehouse ingestion pipeline failing on sales dataset`
3. Set **Top K** to 5 (retrieve top 5 most relevant chunks)
4. Set **Distance threshold** to 0.7 (ignore chunks with less than 70% relevance)
5. Click **Retrieve**
6. Review the returned chunks — confirm they match your runbook content
7. If wrong chunks appear, check whether the runbook files were correctly named and
   whether the content has clear headings so the chunker can split them meaningfully

**Why test retrieval separately:** If retrieval is broken, no amount of prompt tuning
will fix the agent. Always validate the retrieval layer before building the agent on top.

---

#### Step 6 — Set Up Memory Bank

1. Go to **Vertex AI** → **Agent Builder** → **Memory Bank** in the left sidebar
2. Click **Create Memory Bank**
3. Name: `fabric-incident-memory`
4. Description: `Stores ongoing incident context, affected services, and escalation history`
5. Project: `wipro-fabric-data`
6. Region: `us-central1`
7. Click **Create**
8. Note the Memory Bank resource name — you will reference it when configuring the agent

**Why Memory Bank:** Without it, the agent starts from scratch on every follow-up question
in the same incident. An engineer asks "what's the status?" five minutes after reporting
the issue — the agent should remember the incident context, not ask again. Memory Bank
persists that context automatically across turns and sessions.

---

#### Step 7 — Install ADK CLI and Initialise the Agent Project

Run the following in your terminal (not GCP Console — this is on your local machine or a Cloud Shell):

1. Open **Cloud Shell** from the GCP Console top bar (the `>_` icon)
2. Install ADK:
   `pip install google-cloud-aiplatform[adk,agent_engines]`
3. Verify installation:
   `adk --version`
4. Authenticate with your service account:
   `gcloud auth activate-service-account --key-file=PATH_TO_YOUR_KEY_FILE.json`
5. Set default project:
   `gcloud config set project wipro-fabric-data`
6. Set default region:
   `gcloud config set ai/region us-central1`
7. Initialise a new ADK agent project:
   `adk create incident-intelligence-agent`
8. This creates a folder with the agent structure:
   - `agent.py` — where you define the agent behaviour
   - `tools/` — where you define what tools the agent can call
   - `requirements.txt` — agent dependencies
   - `agent.yaml` — agent configuration (model, name, instructions)

**Why ADK CLI over writing raw Python:** ADK CLI scaffolds the correct project structure,
handles packaging for Agent Engine deployment, and manages the local testing runner. It
is the fastest path from idea to deployed agent.

---

#### Step 8 — Configure the Agent in agent.yaml

Open `incident-intelligence-agent/agent.yaml` in Cloud Shell editor and set:

- `name`: `fabric-incident-agent`
- `display_name`: `Fabric Data Platform Incident Assistant`
- `model`: `gemini-2.5-pro`
- `description`: `Answers on-call engineer questions about data pipeline incidents using internal runbooks and post-mortems`
- `instruction`: Set a clear system prompt that tells the agent to:
  - Always search the RAG knowledge base before answering
  - Always cite the specific document name and section it drew from
  - Give remediation steps in numbered list format
  - Always include the escalation path at the end of every response
  - Never invent steps that are not in the retrieved documentation

Under `tools`, add:
- Tool type: `vertex_ai_rag_retrieval`
- Corpus resource name: copy the RAG corpus resource name from Step 4
- Similarity top K: `5`
- Vector distance threshold: `0.7`

Under `memory`:
- Memory Bank resource name: copy from Step 6

Under `generate_content_config`:
- `temperature`: `0.1` — low temperature makes responses more deterministic and safer
  for operational use where you need consistent answers, not creative ones
- `max_output_tokens`: `2048`

**Why temperature 0.1 and not 0:** 0 makes the model repeat the same phrasing verbatim
every time even when slightly different context is retrieved. 0.1 gives minimal variation
while still being near-deterministic. For a runbook agent this is the right balance.

---

#### Step 9 — Test the Agent Locally with ADK CLI

1. In Cloud Shell, navigate to the agent folder:
   `cd incident-intelligence-agent`
2. Run the local ADK test runner:
   `adk run agent.py`
3. This opens an interactive CLI where you can type messages to the agent
4. Test query 1: `Fabric Lakehouse ingestion pipeline is failing on the sales dataset`
5. Verify the response includes: retrieved runbook steps, document citation, escalation path
6. Test query 2: `BigQuery write quota exceeded during evening batch load, how do I fix it`
7. Verify correct runbook is retrieved and steps are accurate
8. Test query 3: Ask a question that has NO matching runbook
9. Verify the agent says it cannot find a relevant runbook rather than making something up
10. Type `exit` to stop the local runner

**Why test locally before deploying:** Local testing is instant and free. Deploying to
Agent Engine takes 3–5 minutes per iteration. Catch prompt and tool configuration issues
locally first.

---

#### Step 10 — Deploy the Agent to Agent Engine

Agent Engine deployment uses the **agents-cli** tool (Google's deployment layer around ADK), not the plain `adk` command.

1. In Cloud Shell, install agents-cli:
   `pip install google-cloud-aiplatform[adk,agent_engines] agents-cli`
2. From the agent project folder, deploy:
   `agents-cli deploy --display-name="fabric-incident-agent" --region=us-central1 --project=wipro-fabric-data`
3. Wait for the command to complete — it packages your agent, registers it with Agent Engine, and returns a resource name like:
   `projects/wipro-fabric-data/locations/us-central1/reasoningEngines/AGENT_ID`
4. Copy this resource name — this is your agent's production endpoint identifier
5. Go to GCP Console → **Vertex AI** → **Agent Builder** → **Deployments**
6. Confirm your agent appears with status **Active**
7. Click the agent → **Test** tab → run the same test queries from Step 9 against the live deployment

**Note on `adk deploy cloud_run`:** If you want to host the agent on Cloud Run instead
of Agent Engine, use `adk deploy cloud_run` from the agent folder. Cloud Run is cheaper
for low-traffic agents; Agent Engine is better for bursty production traffic.

**Why Agent Engine over Cloud Run for this agent:** This agent handles bursty incident
traffic — zero queries at 2 AM, many simultaneous queries during a production incident.
Agent Engine auto-scales from zero to handle the burst without you configuring any
Kubernetes or Cloud Run concurrency settings.

---

#### Step 11 — Connect to Slack (the daily-use interface)

1. Go to **api.slack.com** → Your Apps → **Create New App**
2. Add the **Incoming Webhooks** feature → enable it → add to your `#data-incidents` channel
3. Copy the Webhook URL
4. Go to GCP Console → **Secret Manager** → **Create Secret**
   - Name: `slack-webhook-url`
   - Value: paste the Slack webhook URL
   - Click **Create**
5. Go to **Cloud Run** → **Create Service**
6. Container image: use the **inline editor** and create a simple HTTP server that:
   - Accepts a POST from Slack
   - Calls your Agent Engine endpoint with the message text
   - Posts the agent response back to Slack via the webhook URL
7. Set the service account to `fabric-agent-sa`
8. Region: `us-central1`
9. Allow unauthenticated invocations (Slack webhooks cannot carry GCP auth headers)
10. Deploy the Cloud Run service and copy its public URL
11. In Slack app settings → **Slash Commands** → **Create New Command**
    - Command: `/ops`
    - Request URL: paste your Cloud Run service URL
    - Description: `Ask the data platform incident agent`
12. Test in Slack: type `/ops Fabric pipeline failing on finance dataset`

---

#### Production Considerations for Scenario 1

- **Corpus freshness:** Schedule a weekly **Cloud Scheduler** job to re-import updated
  GCS files into the RAG corpus. Go to Cloud Scheduler → Create Job → target: Cloud Run
  function that calls the RAG Engine import API.
- **Evaluation:** Build a golden test set of 50 past incident questions with known correct
  answers. Run it weekly using the ADK local runner in a CI job. Alert if accuracy drops.
- **Cost monitoring:** Each RAG query costs approximately $0.002–$0.005. At 200 queries
  per day that is under $30 per month. Go to **Billing** → **Budgets & Alerts** → set
  an alert at $50 per month for this project.
- **Access control:** Only the SRE and data platform team service accounts should be able
  to query the RAG corpus. Go to RAG Engine → your corpus → **Permissions** → restrict
  to named service accounts only.

---

## Scenario 2: Multi-Agent CI/CD & Data Pipeline Review System
### (ADK ParallelAgent + MCP + BigQuery + Agent Engine + Pub/Sub)

### The Story (tell this in interviews)

> "At Wipro on the Fabric data client project, every PR touching infrastructure or data
> pipeline definitions had to be manually reviewed for security issues, cost impact, and
> data governance compliance before merge. This was taking 2–3 hours of senior engineer
> time per PR. I built a multi-agent pipeline using ADK that runs automatically on every
> PR: a Security Agent scans for credential exposure and IAM over-permissioning, a Cost
> Agent estimates the monthly GCP spend delta using BigQuery billing data, and a
> Compliance Agent checks against our data governance and SOC 2 policy checklist.
> An Orchestrator runs all three in parallel, merges their reports, and posts a
> structured review comment to GitHub — all within 90 seconds of the PR opening.
> We cut infrastructure PR review time from 3 hours to 15 minutes of human review."

---

### Services Used & Why

| Service | Role | Why This One |
|---|---|---|
| **ADK ParallelAgent** | Orchestration | Run 3 review agents at the same time, not one after another |
| **Gemini 2.5 Flash** | Agent LLM | Flash is 5x cheaper and faster than Pro; sufficient for structured analysis |
| **BigQuery MCP Server** | Cost data | GCP billing export lives in BigQuery — MCP gives the agent direct query access |
| **GitHub MCP Server** | PR access | Reads PR diff and posts comments without custom GitHub API wrappers |
| **Agent Engine** | Runtime | Auto-scales for bursty PR webhook traffic |
| **Cloud Run** | Webhook receiver | Receives GitHub webhook and publishes to Pub/Sub asynchronously |
| **Cloud Pub/Sub** | Decoupling | GitHub requires a response in under 10 seconds; agents take 30–90 seconds |
| **Secret Manager** | Credentials | GitHub token and GCP service account key stored securely |
| **Cloud Trace** | Debugging | Shows exactly which agent found what and how long each agent took |

---

### Multi-Agent Architecture (no code — visual flow)

```
GitHub PR Opened
        ↓
Cloud Run (webhook receiver)
        ↓ publishes event message instantly
Cloud Pub/Sub topic: pr-review-requests
        ↓ triggers asynchronously
Orchestrator Agent (ADK ParallelAgent)
        ↓ fans out simultaneously to all three
┌───────────────────┬───────────────────┬────────────────────┐
Security Agent      Cost Agent          Compliance Agent
(checks for         (queries BigQuery    (checks SOC 2 and
credentials,        billing, estimates   data governance
over-permissioned   monthly cost delta   controls: encryption,
IAM roles,          from Terraform       audit logging,
public buckets)     changes)             access controls)
└───────────────────┴───────────────────┴────────────────────┘
        ↓ all three complete, results merged by orchestrator
GitHub MCP Server → Post structured review comment on PR
```

---

### Step-by-Step: Building the Multi-Agent PR Review Pipeline

---

#### Step 1 — Set Up BigQuery Billing Export

1. Go to GCP Console → **Billing** → select your billing account
2. Click **Billing Export** → **BigQuery Export** tab
3. Click **Edit Settings** under Standard Usage Cost
4. Select project: `wipro-fabric-data`
5. Dataset name: `billing_export`
6. Click **Save** — GCP will start exporting daily billing data to BigQuery
7. Wait 24 hours for the first export to populate (this is a known delay on first setup)
8. Go to **BigQuery** → your project → `billing_export` dataset → confirm the
   `gcp_billing_export_v1_XXXXXX` table exists with data

**Why BigQuery billing export:** This is the most granular, queryable source of GCP cost
data. Your Cost Agent can query it to say "this Terraform change adds a new Cloud SQL
instance — based on the current billing data for similar instances in this project, it
will cost approximately $180/month."

---

#### Step 2 — Deploy the BigQuery MCP Server

1. Go to GCP Console → **Cloud Run** → **Create Service**
2. Container image: `gcr.io/cloudrun/mcp-bigquery:latest`
   (Google's pre-built BigQuery MCP server image)
3. Service name: `mcp-server-bigquery`
4. Region: `us-central1`
5. Under **Variables & Secrets** → add environment variable:
   - `PROJECT_ID` = `wipro-fabric-data`
   - `DATASET_ID` = `billing_export`
6. Service account: `fabric-agent-sa`
7. Authentication: **Require authentication** (only your agents can call it)
8. Click **Deploy**
9. Copy the service URL once deployed — you will reference this in the agent config

---

#### Step 3 — Deploy the GitHub MCP Server

1. Go to **GitHub** → **Settings** → **Developer Settings** → **Personal Access Tokens**
2. Generate a new token with scopes: `repo` (read PR diffs, write comments)
3. Copy the token
4. Go to GCP Console → **Secret Manager** → **Create Secret**
   - Name: `github-mcp-token`
   - Value: paste the GitHub token
   - Click **Create**
5. Go to **Cloud Run** → **Create Service**
6. Container image: `gcr.io/cloudrun/mcp-github:latest`
7. Service name: `mcp-server-github`
8. Region: `us-central1`
9. Under **Variables & Secrets** → add secret reference:
   - Environment variable name: `GITHUB_TOKEN`
   - Secret: `github-mcp-token` → latest version
10. Service account: `fabric-agent-sa`
11. Authentication: **Require authentication**
12. Click **Deploy** and copy the service URL

---

#### Step 4 — Create Three ADK Specialist Agent Projects

In Cloud Shell, create three separate agent projects:

**Security Agent:**
- `adk create security-reviewer-agent`
- Open `agent.yaml` and configure:
  - Model: `gemini-2.5-flash`
  - Instruction: Review the PR diff for hardcoded secrets and API keys, IAM roles
    broader than least-privilege (owner/editor roles), public-facing storage buckets
    without business justification, unencrypted data stores, missing VPC service controls.
    Return a structured finding with severity HIGH/MEDIUM/LOW and whether this is
    BLOCKING (must fix before merge) or WARNING (should fix but not blocking).
  - Tools: add the GitHub MCP server URL from Step 3

**Cost Agent:**
- `adk create cost-analyzer-agent`
- Open `agent.yaml` and configure:
  - Model: `gemini-2.5-flash`
  - Instruction: Analyse the Terraform diff for new or modified GCP resources. Query
    the BigQuery billing export to find current monthly costs for similar resources
    already running in this project. Estimate the monthly cost delta from this change.
    Flag any single-resource change adding more than $500/month as HIGH_COST.
    Return a cost breakdown per resource with current cost, estimated new cost, and delta.
  - Tools: add both the BigQuery MCP server URL and the GitHub MCP server URL

**Compliance Agent:**
- `adk create compliance-checker-agent`
- Open `agent.yaml` and configure:
  - Model: `gemini-2.5-flash`
  - Instruction: Check this infrastructure change against Wipro Fabric data client's
    SOC 2 Type II controls: all data stores must have encryption at rest and in transit,
    no direct internet ingress without WAF, all service accounts must have key rotation
    under 90 days, audit logging must be enabled on all new GCP projects, no service
    accounts with primitive roles (Owner or Editor). Return compliant: true/false and
    a list of any violations found.
  - Tools: add the GitHub MCP server URL

---

#### Step 5 — Create the Orchestrator Agent Project

1. In Cloud Shell: `adk create pr-review-orchestrator`
2. Open `agent.yaml` and configure:
   - Agent type: `ParallelAgent`
   - Sub-agents: list the three specialist agents from Step 4
   - Model: `gemini-2.5-flash`
   - Instruction: Run all three sub-agents in parallel. Once all complete, merge their
     outputs into a single structured GitHub PR comment in Markdown format with sections
     for Security, Cost Impact, Compliance, and Action Required. Post the comment using
     the GitHub MCP tool. Be direct — engineers are busy.
   - Tools: GitHub MCP server URL (for posting the final comment)

**Why ParallelAgent:** Running three agents sequentially would take 45–60 seconds.
Running them in parallel takes 15–20 seconds total — the overall time is the slowest
of the three, not the sum of all three. For a PR review experience that feels instant,
parallel execution is essential.

---

#### Step 6 — Set Up Cloud Pub/Sub Topic

1. Go to GCP Console → **Pub/Sub** → **Topics** → **Create Topic**
2. Topic ID: `pr-review-requests`
3. Leave default settings (standard topic, not FIFO)
4. Click **Create**
5. Click the topic → **Subscriptions** tab → **Create Subscription**
6. Subscription ID: `pr-review-agent-sub`
7. Delivery type: **Push**
8. Endpoint URL: leave blank for now — you will fill this after deploying agents in Step 8

**Why Pub/Sub and not direct webhook to agent:** GitHub sends a webhook and expects a
response in under 10 seconds or it marks the delivery as failed and retries. Your agent
takes 30–90 seconds. Pub/Sub absorbs the event instantly, returns 200 to GitHub, and
delivers the message to the agent at its own pace. This decouples webhook latency from
agent processing time.

---

#### Step 7 — Create the GitHub Webhook Receiver in Cloud Run

1. Go to **Cloud Run** → **Create Service**
2. Container: use **Write your own code** option in Cloud Shell inline editor
3. Create a minimal HTTP service that:
   - Listens on PORT (Cloud Run sets this automatically)
   - Accepts POST requests at `/github-webhook`
   - Reads the JSON body from GitHub
   - Checks if the event action is `opened` or `synchronize` and the event is `pull_request`
   - Publishes the full payload to the Pub/Sub topic `pr-review-requests`
   - Returns HTTP 200 immediately
4. Build the container: `gcloud builds submit --tag gcr.io/wipro-fabric-data/github-webhook-receiver`
5. Deploy: `gcloud run deploy github-webhook-receiver --image gcr.io/wipro-fabric-data/github-webhook-receiver --region us-central1 --allow-unauthenticated`
6. Copy the Cloud Run service URL

**Why allow unauthenticated on this one service:** GitHub cannot attach a GCP auth
header to its webhook. This is the only public-facing entry point. Validate GitHub's
webhook secret header inside the function to confirm the request is genuinely from GitHub.

---

#### Step 8 — Deploy All Agents to Agent Engine

For each of the four agents (security, cost, compliance, orchestrator):

1. In Cloud Shell, navigate to the agent folder
2. Run: `agents-cli deploy --display-name="AGENT_NAME" --region=us-central1 --project=wipro-fabric-data`
3. Wait for the deployment to complete and note each agent's resource name
4. Go to GCP Console → **Vertex AI** → **Agent Builder** → **Deployments**
5. Confirm all four agents show status **Active**

Then wire up Pub/Sub:
6. Go to **Pub/Sub** → **Subscriptions** → `pr-review-agent-sub` → **Edit**
7. Set the Push endpoint URL to the orchestrator agent's invocation endpoint
8. Save

---

#### Step 9 — Configure GitHub Webhook

1. Go to your GitHub repository → **Settings** → **Webhooks** → **Add webhook**
2. Payload URL: paste the Cloud Run webhook receiver URL from Step 7
3. Content type: `application/json`
4. Secret: generate a random string, set it here and store it in Secret Manager
5. Events: select **Pull requests** only
6. Click **Add webhook**
7. GitHub will send a test ping — verify it appears in Cloud Run logs

---

#### Step 10 — End-to-End Test

1. Open a test PR in your GitHub repository that modifies a Terraform file
2. Watch the GitHub webhook fire in **Cloud Run** → **Logs**
3. Watch the message arrive in **Pub/Sub** → **Metrics** → message count graph
4. Watch all four agents execute in **Vertex AI** → **Deployments** → each agent's **Logs**
5. Watch the structured review comment appear on your GitHub PR within 90 seconds
6. Go to **Cloud Trace** → search by time — see the full parallel execution span showing
   all three specialist agents running simultaneously

---

#### Production Considerations for Scenario 2

- **Caching:** The same PR commit SHA should not trigger a re-review. Go to **Firestore**
  → create a collection `pr-review-cache` with SHA as the document key and TTL of 24 hours.
  Check this in the webhook receiver before publishing to Pub/Sub.
- **False positive tracking:** Add a :thumbsup: / :thumbsdown: reaction to the GitHub
  comment. Log reactions via GitHub webhook. After 4 weeks, review which findings engineers
  consistently override — update the agent instruction to stop flagging those patterns.
- **Cost per PR review:** Each review costs approximately $0.01–$0.03 in LLM tokens plus
  the BigQuery query cost. At 100 PRs per day that is $30–90 per month. Set a budget alert.

---

## Scenario 3: Cross-Platform Data Ops Assistant Using A2A Protocol
### (A2A + ADK + Agent Engine + Memory Bank + Cloud Run)

### The Story (tell this in interviews)

> "After building the first two agents, teams across Wipro's Fabric data client project
> started asking if their tools could join our agent ecosystem. The ITSM team had their
> own ServiceNow agent. The data governance team had a Collibra agent. The monitoring
> team had a PagerDuty agent. They were all built independently, on different platforms,
> in different languages. I used the A2A protocol to wire them together. Now our
> orchestrator agent — running on GCP Agent Engine — can take a natural language command
> like 'Create a P2 incident for the Fabric Lakehouse pipeline failure on the finance
> dataset, pull the last 3 related incidents from ServiceNow, check data lineage in
> Collibra, and post a status update to #data-incidents in Slack' and execute across
> four different agent platforms in one shot, all authenticated, all logged, with a
> full audit trail."

---

### Services Used & Why

| Service | Role | Why This One |
|---|---|---|
| **ADK (LlmAgent as orchestrator)** | Central coordinator | Interprets intent, decides which A2A agents to call |
| **A2A Protocol** | Cross-agent communication | ServiceNow/PagerDuty/Collibra agents live on external platforms |
| **Agent Card** | Agent discovery | Each external agent declares its skills at `/.well-known/agent.json` |
| **Agent Engine** | Orchestrator hosting | Managed production runtime for the central ADK agent |
| **Memory Bank** | Incident context | Remembers ongoing incidents across follow-up questions |
| **Cloud Run** | Internal A2A agents | Hosts lightweight GCP-native skill agents (log reader, cost checker) |
| **Cloud Logging** | Audit trail | Every A2A call logged — required for SOC 2 compliance |
| **Secret Manager** | OAuth tokens | A2A OAuth tokens stored and rotated here, never in code |
| **Gemini 2.5 Pro** | Orchestrator LLM | Complex multi-step reasoning over ambiguous natural language input |
| **Cloud Logging MCP** | GKE/pipeline log access | Reads pipeline error logs via the pre-built Cloud Logging MCP server |

---

### Step-by-Step: Building the A2A Cross-Platform Ops Assistant

---

#### Step 1 — Understand Agent Cards Before Building

An Agent Card is a JSON file that every A2A-compliant agent publishes at the path
`/.well-known/agent.json` on its server URL. Before building anything, go to the
external agents your team uses and check if they already publish an Agent Card:

1. Open your browser and navigate to: `https://YOUR-SERVICENOW-AGENT-URL/.well-known/agent.json`
2. If it returns a JSON document, that agent is already A2A-compliant
3. If it returns 404, that agent is not yet A2A-compliant — you will need to wrap it

The Agent Card contains:
- `name` and `description` — what the agent is
- `skills` — a list of tasks this agent can perform, each with an ID, description,
  input format, and output format
- `authentication` — which OAuth 2.0 flows are supported and which scopes are needed

**Why this matters for interviews:** Being able to explain Agent Card discovery is what
separates someone who deployed a toy A2A demo from someone who integrated real enterprise
agents. Interviewers will ask how your orchestrator knows what the external agent can do.

---

#### Step 2 — Create OAuth 2.0 Credentials for A2A Authentication

For each external agent your orchestrator will call, you need OAuth credentials:

1. Go to GCP Console → **APIs & Services** → **Credentials** → **Create Credentials**
2. Select **OAuth client ID**
3. Application type: **Web application**
4. Name: `a2a-servicenow-client` (one per external agent)
5. Click **Create** — copy the Client ID and Client Secret
6. Go to **Secret Manager** → **Create Secret**
   - Name: `a2a-servicenow-client-secret`
   - Value: paste the Client Secret
   - Click **Create**
7. Repeat for each external agent (PagerDuty, Collibra, Slack)

**Why Secret Manager and not environment variables:** OAuth secrets must be rotated.
Secret Manager supports automatic rotation and versioning. Environment variables in
Cloud Run or Agent Engine cannot be rotated without redeploying the agent. For A2A
authentication tokens that may need emergency revocation, Secret Manager is the only
safe option.

---

#### Step 3 — Create an Internal A2A Skill Agent (Cloud Logging Reader)

This is a GCP-native agent that reads Cloud Logging data and exposes it via A2A.
It will be hosted on Cloud Run and called by the orchestrator.

1. Go to **Cloud Shell**
2. Create a new ADK agent project: `adk create log-reader-agent`
3. In `agent.yaml` configure:
   - Model: `gemini-2.5-flash`
   - Instruction: Query Cloud Logging for error logs from a given service or pipeline
     within a given time window. Return the top 10 error messages, their frequency,
     and the first occurrence timestamp.
   - Tools: Cloud Logging MCP server
4. This agent needs to publish an Agent Card at `/.well-known/agent.json`
5. Create a static JSON file in the project with the Agent Card content:
   - name: `Cloud Logging Reader Agent`
   - skills: one skill with id `query_logs`, description `Query Cloud Logging for errors from a pipeline or service`
   - authentication: your GCP OAuth 2.0 client credentials
6. Build the container: `gcloud builds submit --tag gcr.io/wipro-fabric-data/log-reader-agent`
7. Deploy to Cloud Run: `gcloud run deploy log-reader-agent --image gcr.io/wipro-fabric-data/log-reader-agent --region us-central1 --no-allow-unauthenticated`
8. Verify the Agent Card is reachable: `curl https://YOUR-LOG-READER-URL/.well-known/agent.json`

---

#### Step 4 — Verify External Agent Cards

For each external agent (ServiceNow, PagerDuty, Collibra, Slack):

1. Fetch their Agent Card: `curl https://AGENT-URL/.well-known/agent.json`
2. Review the `skills` array — note each skill's `id` and expected `input` format
3. Review the `authentication` section — confirm it uses `oauth2` with `clientCredentials` flow
4. Note the `tokenUrl` — this is where your orchestrator will exchange credentials for an access token
5. Create a reference document listing all external agents, their skill IDs, and token URLs

**Why document this manually:** A2A orchestrators need to know which skill ID to send to
which agent. If the external team changes a skill ID and you have not read the updated
Agent Card, your orchestrator will start getting 404 errors. The Agent Card is the
contract — check it when things break.

---

#### Step 5 — Configure the A2A Orchestrator Agent

1. In Cloud Shell: `adk create a2a-ops-orchestrator`
2. In `agent.yaml` configure:
   - Model: `gemini-2.5-pro` (needed for complex multi-step reasoning across ambiguous input)
   - Memory Bank: reference the `fabric-incident-memory` from Scenario 1
   - Instruction: You are the Data Operations AI for Wipro's Fabric data client.
     When an engineer reports a data pipeline incident or asks for operational help,
     follow this order:
     1. Query the Cloud Logging Reader agent for errors in the affected pipeline
     2. Search ServiceNow via the ServiceNow agent for past similar incidents
     3. Check data lineage in Collibra if the issue may be schema or data quality related
     4. Create a PagerDuty incident if severity is P1 or P2
     5. Post a summary to #data-incidents in Slack
     Use Memory Bank context from this session and previous sessions about this incident.
     Be concise — engineers in incidents need facts, not explanations.

3. Under `tools`, add A2A tool definitions for each external agent:
   - Tool name: `query_pipeline_logs` → points to Cloud Logging Reader Agent, skill: `query_logs`
   - Tool name: `search_past_incidents` → points to ServiceNow Agent, skill: `search_incidents`
   - Tool name: `check_data_lineage` → points to Collibra Agent, skill: `get_lineage`
   - Tool name: `create_incident` → points to PagerDuty Agent, skill: `create_incident`
   - Tool name: `notify_slack` → points to Slack Agent, skill: `post_message`

4. For each A2A tool, configure:
   - `agent_url`: the external agent's base URL
   - `skill_id`: the specific skill to invoke
   - `auth_secret`: the Secret Manager path for this agent's OAuth credentials
   - `timeout_seconds`: 30 (if the external agent takes longer, proceed without it)

---

#### Step 6 — Deploy the Orchestrator to Agent Engine

1. In Cloud Shell, navigate to the orchestrator folder
2. Run: `agents-cli deploy --display-name="a2a-ops-orchestrator" --region=us-central1 --project=wipro-fabric-data`
3. Wait for deployment to complete
4. Go to GCP Console → **Vertex AI** → **Deployments** → confirm status **Active**
5. Note the orchestrator's invocation endpoint URL

---

#### Step 7 — Test the Full A2A Flow End to End

1. Go to **Vertex AI** → **Deployments** → click your orchestrator → **Test** tab
2. Send a test message: `Fabric Lakehouse ingestion pipeline failing on finance dataset since 2 PM`
3. Watch in **Cloud Trace** → you should see:
   - Orchestrator receives message → calls `query_pipeline_logs` tool
   - A2A call goes to Cloud Logging Reader agent → returns error summary
   - Orchestrator calls `search_past_incidents` → A2A call to ServiceNow agent → returns past incidents
   - Orchestrator calls `create_incident` → A2A call to PagerDuty agent → incident created
   - Orchestrator calls `notify_slack` → A2A call to Slack agent → message posted
4. Go to **Cloud Logging** → filter by `resource.type="aiplatform.googleapis.com/DeployedModel"`
5. Confirm you see structured log entries for every A2A outbound call with: agent name,
   skill called, response status, and latency
6. Check #data-incidents in Slack — confirm the formatted incident update appeared

---

#### Step 8 — Enable SOC 2 Audit Logging for A2A

1. Go to **Cloud Logging** → **Log Router** → **Create Sink**
2. Sink name: `a2a-audit-sink`
3. Destination: **Cloud Storage bucket** → create a new bucket `wipro-fabric-audit-logs`
4. Filter: include all logs from the orchestrator agent's resource name
5. Click **Create**
6. Go to **Cloud Storage** → `wipro-fabric-audit-logs` → **Lifecycle** → **Add Rule**
7. Set retention: 90 days (SOC 2 minimum requirement for audit log retention)
8. Enable **Object Versioning** on the bucket — this makes the logs tamper-evident

**Why this step:** SOC 2 auditors will ask: what did your AI agent do at 3 AM on a
specific date, what external systems did it call, and what data did it send? Without
this sink, logs are retained in Cloud Logging for only 30 days and are not in an
immutable store. This bucket gives you a 90-day tamper-evident audit trail.

---

#### Production Considerations for Scenario 3

- **A2A timeout handling:** Each external A2A agent has a 30-second timeout configured.
  If an agent times out, the orchestrator continues with the other agents and notes the
  gap in the response. Go to Cloud Logging and alert when timeout rate for any external
  agent exceeds 10% — that signals the external agent has a performance problem.
- **Circuit breaker pattern:** If PagerDuty agent fails 3 times within 5 minutes,
  stop calling it automatically and alert the on-call team to check PagerDuty directly.
  Implement this as a Firestore counter — the orchestrator checks it before each call.
- **Token rotation for A2A OAuth:** Go to Secret Manager → each A2A secret → enable
  **Automatic Rotation** with a 30-day rotation period. Create a Cloud Function that
  the rotation trigger calls to exchange credentials and update the secret with a new token.
- **Agent Card version monitoring:** Create a Cloud Scheduler job that daily fetches
  each external Agent Card and compares it to the last known version stored in Firestore.
  If a skill ID changed or an authentication scheme changed, send an alert to your team
  before your orchestrator starts failing silently.

---

## Part 3: Interview Questions & Answers

### Section A: Foundational GenAI & RAG

**Q1. What is RAG and why do you use it instead of fine-tuning?**

RAG retrieves relevant documents at inference time and injects them into the LLM prompt.
Fine-tuning bakes knowledge into the model weights during training. For operational data
— runbooks, pipeline schemas, incident reports — that changes weekly or monthly, RAG is
the right choice. You update the corpus without touching the model. Fine-tuning is for
teaching the model a fixed reasoning style or domain vocabulary. In data platform
operations, almost always start with RAG. Only fine-tune after proving RAG's retrieval
quality has hit a ceiling that no prompt tuning can overcome.

**Q2. How do you evaluate a RAG pipeline in production?**

Three metrics: retrieval precision — are the top-K chunks actually relevant to the
question? Answer faithfulness — does the response only use what was retrieved, or is
the model adding things from its training data? Answer relevance — does the response
actually answer what was asked? Build a golden evaluation set of 50 question-answer
pairs where you know the correct answer and source document. Run this weekly in a CI job.
If retrieval precision drops, check whether new documents changed the chunking quality.
If faithfulness drops, strengthen the system prompt to cite sources more strictly.

**Q3. What is chunk overlap in RAG and why does it matter?**

When a document is split into chunks, overlap means the last N tokens of chunk 1 are
repeated at the start of chunk 2. Without overlap, a sentence spanning a chunk boundary
is truncated in both chunks, losing meaning. With overlap, both chunks carry enough
context to be retrievable by relevant queries. Too much overlap wastes embedding space
and increases retrieval noise. Standard is 10–20% of chunk size. For runbooks with steps
that reference prior steps, 100-token overlap on 512-token chunks is the right balance.

**Q4. What is grounding and how is it different from RAG?**

Grounding is the higher-level goal: ensuring the LLM response is anchored to a
verifiable source. RAG is one way to achieve grounding using your private document corpus.
Google Search Grounding is another way using live public web results. In Vertex AI you
can layer both: ground on your internal corpus first, fall back to Google Search for
public facts the corpus does not cover. Grounding gives you citations. Citations give
you auditability. Auditability is a mandatory requirement for any regulated enterprise
deployment.

---

### Section B: ADK & Agent Architecture

**Q5. What are the main agent types in ADK and when do you choose each?**

LlmAgent is the base type for any task needing LLM reasoning, tool use, or natural
language understanding — the default choice for almost everything. SequentialAgent
is for pipelines where each step depends on the previous step's output, like ingest
then analyse then report. ParallelAgent is for independent tasks where you want to
minimise total time — running security, cost, and compliance review simultaneously
is a classic example. LoopAgent is for polling or retry workflows — keep checking
the status of a deployment until it reaches SUCCEEDED or FAILED.

**Q6. How do you handle agent state across a multi-turn conversation?**

ADK's session service maintains conversation history within a session automatically.
For context that needs to persist across sessions — things the agent should remember
next week — you use Memory Bank. The pattern is: at the end of each session, extract
key facts (incident ID, affected pipeline, decisions made, steps already tried) and
write them to Memory Bank. At the start of the next session, retrieve relevant memories
and inject them into the context. This gives continuity without blowing up the context
window by replaying full conversation history.

**Q7. How do you prevent an agent from hallucinating tool calls?**

Three controls. First: tool descriptions must be precise and specific. A vague
description like "useful for looking things up" causes the agent to call the tool at
wrong times. A precise description like "Query Cloud Logging for pipeline errors within
a specific time window" makes the agent call it only when appropriate. Second: use low
temperature (0.1 or below) for agents that call real-world APIs. Higher temperature
increases creative variation which is harmful when you need deterministic tool selection.
Third: validate tool inputs before execution — check that required parameters are
present and in the right format before making the actual API call.

**Q8. How do you test ADK agents before deploying to production?**

Unit test each tool function independently — no LLM involved, just call the function
with test inputs and assert on outputs. Integration test each agent using the ADK local
runner (`adk run`) with a fixed set of test messages — assert on which tools were called,
in what order, with what arguments. End-to-end test on a staging Agent Engine deployment
with synthetic but realistic data. Set up continuous evaluation against a golden question
set in a CI pipeline. Agents can degrade silently when the underlying model version
changes — continuous evals are the only reliable way to detect this.

---

### Section C: Multi-Agent & A2A

**Q9. What is the A2A protocol and why does it exist?**

A2A is an open HTTP standard for agent-to-agent communication across platforms. It exists
because enterprise environments have heterogeneous agent ecosystems. A GCP agent needs
to call a ServiceNow agent, a Salesforce agent, a Collibra agent — all built by different
teams on different platforms. Without A2A, every pair of agents needs a custom integration.
A2A standardises four things: agent capability discovery via Agent Cards, task submission
via JSON-RPC, streaming results via Server-Sent Events, and mutual authentication via
OAuth 2.0 and JWT. As of June 2026, it is governed by the Linux Foundation and in
production use at 150+ enterprises.

**Q10. What is an Agent Card and what does it contain?**

An Agent Card is a JSON document published at `/.well-known/agent.json` on every
A2A-compliant agent server. It declares: the agent's name, URL, and version; a list of
skills it can perform with the ID, description, accepted input formats, and output
formats for each; and the authentication scheme required. It is the agent equivalent
of an OpenAPI specification. Orchestrators read the Agent Card to discover what an
external agent can do without any custom documentation or integration code. When the
external team adds a new skill, it appears in the Agent Card automatically — the
orchestrator discovers it on the next card fetch without code changes.

**Q11. How does authentication work in A2A?**

A2A uses OAuth 2.0 client credentials flow. The orchestrator agent has its own identity
as a GCP service account. When calling an external A2A agent, it exchanges its client
credentials (stored in Secret Manager) for a short-lived access token scoped to specific
skills. The access token is attached to the Authorization header of the A2A task request
as a JWT. The external agent validates the JWT — checking the issuer, audience, expiry,
and scope — before processing the task. No credentials ever appear in the request body
or task payload. Tokens are cached in memory and refreshed before expiry.

**Q12. How do you handle failures in a multi-agent pipeline?**

Three patterns. First, independent failure: in a ParallelAgent, a failed Cost Agent
should not block the Security Agent result. Each specialist agent wraps its logic in
error handling and returns a structured failure response that the orchestrator can include
in the report as "Cost analysis unavailable." Second, circuit breaker: if an external
A2A agent fails more than 3 times within 5 minutes, stop calling it and alert your team
instead of hammering a failing external service. Implement this as a counter in Firestore.
Third, idempotency: agents that create external resources — PagerDuty incidents, Jira
tickets, ServiceNow records — must check whether the resource already exists before
creating it to prevent duplicates on retry.

---

### Section D: Production, Observability & Cost

**Q13. How do you monitor an agentic AI system in production?**

Three layers: correctness, performance, and cost. For correctness: run continuous evals
against a golden dataset, alert if accuracy drops more than 5% week-over-week. For
performance: use Cloud Trace to track p50, p95, and p99 latency for every tool call and
LLM call separately — a slow tool is usually the bottleneck, not the LLM. For cost: track
input and output token counts per agent turn in Cloud Logging, export to BigQuery, and
build a Looker Studio dashboard showing cost per agent per day. The most important metric
that nobody tracks at first: agent turn completion rate — how often does the agent
successfully complete the task in one turn versus requiring follow-up clarification? Low
completion rate means the agent is misunderstanding the request.

**Q14. What is prompt versioning and why does it matter for agents in production?**

Your agent's system prompt is code. It determines agent behaviour as much as any
function in your application. It must be version-controlled in Git, reviewed before
merging, tested against the eval suite before deployment, and rolled back if it causes
a regression. Store prompts in a configuration file, not as inline strings in the agent
code. Tag every LLM call in Cloud Logging with the prompt version identifier so you can
correlate behaviour changes to specific prompt changes. A prompt change that silently
degraded agent quality two weeks ago is very hard to diagnose without this trace.

**Q15. How do you control and forecast costs for a production agentic AI system?**

Instrument every LLM call with input token count, output token count, and model name,
and write these as structured JSON fields in Cloud Logging. Create a BigQuery log sink
so all agent logs flow into a BigQuery table. Build a Looker Studio dashboard with daily
cost per agent, cost per tool type, and week-over-week trend. Set a billing budget alert
at 80% of your monthly cap. For cost reduction: use Gemini Flash instead of Pro for
structured, deterministic tasks like compliance checking — Flash is 5x cheaper with
acceptable quality for rule-based analysis. Cache repeated queries in Firestore with a
15-minute TTL — if 30% of your queries ask the same question (like "who is on call?"),
caching saves 30% of token costs.

**Q16. How do you implement SOC 2 compliance for an AI agent deployment?**

Four controls. First, data retention: log every agent input and output for 90 days with
the identity of who triggered it. Use a Cloud Logging sink to a versioned, immutable GCS
bucket. Second, access control: agent service accounts must follow least-privilege — each
agent has only the IAM roles it needs for its specific tools. No agent service account
has Editor or Owner role. Third, audit trail: every tool call, A2A call, and LLM call
is logged as a structured entry with timestamp, acting identity, action taken, and
result. Fourth, data residency: configure Vertex AI to process data in the required
region — for regulated clients, specify `europe-west1` or the contractually required
region in every API call and agent configuration.

---

### Section E: Architecture Decisions

**Q17. When do you choose RAG Engine over building a custom RAG pipeline?**

RAG Engine for 80% of production use cases. It handles chunking, embedding model
management, vector storage, and retrieval with a managed API, and you pay per query
rather than running infrastructure. Build a custom pipeline when you need a fine-tuned
embedding model trained on your specific domain vocabulary, when you need hybrid search
combining BM25 keyword search with semantic search, or when you need to integrate an
existing Elasticsearch or Pinecone index that you cannot migrate. Custom RAG is 3–5x
more engineering effort — only justified when RAG Engine has proven to be the bottleneck.

**Q18. When do you deploy agents on Agent Engine versus GKE?**

Agent Engine for the majority of production use cases: zero-infrastructure management,
built-in auto-scaling, integrated Cloud Trace and Cloud Logging, and deployment via a
single ADK CLI command. Choose GKE when you need GPU-attached nodes for running local
model inference alongside the agent, when the agent must run in a private VPC with no
public internet access and Agent Engine's networking model cannot accommodate this, or
when you need to co-locate the agent with other Kubernetes workloads to reduce latency
on shared infrastructure.

**Q19. How do you design a multi-agent system that remains maintainable over 12 months?**

Three principles. First, single responsibility: each agent does exactly one thing well.
An agent that does security review, cost estimation, and compliance checking will be
impossible to debug when one of those functions breaks. Second, contract-first design:
define each agent's input and output schema before writing the agent logic. Write these
as explicit data structures. When the agent's internal logic changes, the schema contract
protects all callers from breaking changes. Third, observability from day one: build
Cloud Trace instrumentation and structured logging into every agent before the first
deployment. Adding observability later to a running production agent system requires
redeployment of every agent and often reveals months of invisible problems.

---

### Section F: Quick-Fire Scenario Questions

**Q20. Your RAG agent's answer quality degrades over two weeks. What do you investigate?**

Check corpus freshness first — were new documents added that have different formatting or
structure that breaks the chunker? Next, check whether the embedding model was silently
updated to a new version — Google occasionally releases new default embedding model
versions. Then run the golden eval set to identify which question categories degraded.
If retrieval precision dropped, the issue is in chunking or document quality. If answer
faithfulness dropped, the model may be ignoring retrieved context — strengthen the
system prompt instruction to explicitly require citing only provided sources.

**Q21. An A2A agent you depend on is returning 50% errors. What is your debugging process?**

Fetch their current Agent Card and compare it to your saved version. Has a skill ID
changed? Has the authentication scheme changed? If the Agent Card is identical, examine
the error responses: 401 errors indicate an authentication problem, likely token expiry
or rotation — refresh the OAuth token from Secret Manager. 429 errors indicate rate
limits — add exponential backoff. 5xx errors mean the external agent has a server-side
problem — activate your circuit breaker and notify the team that owns that agent.
Always log the full request payload alongside the error so you can give the external
team a reproduction case.

**Q22. A new external team says their agent supports A2A. How do you onboard it?**

First, fetch and review their Agent Card at `/.well-known/agent.json`. Confirm the skill
IDs match what they claim and the authentication scheme is OAuth 2.0 client credentials.
Request test credentials from them for their staging environment. Validate that you can
fetch an OAuth token, submit a task using the correct skill ID, and receive a valid
response. If that works, store their production credentials in Secret Manager. Add an
A2A tool definition to your orchestrator's configuration referencing their agent URL,
skill IDs, and credential secret path. Update the orchestrator system prompt to describe
when to use this new tool. Deploy the updated orchestrator to Agent Engine staging.
Run your full eval set. Deploy to production only after staging evals pass.

**Q23. How do you explain MCP versus A2A to a non-technical stakeholder?**

MCP is about giving an AI agent access to tools and data — like giving a person a
keyboard, a database login, and access to your file system. It connects the agent to
the data it needs to work. A2A is about AI agents delegating work to other AI agents —
like an employee who can ask a specialist colleague on another team to handle part of
a task. MCP is vertical: agent down to tools. A2A is horizontal: agent across to other
agents. You need both in a complete enterprise agentic system.

**Q24. What would you build first at a company that wants to add AI to its data platform?**

Start with a RAG agent on top of existing runbooks and data dictionary documentation.
It is the highest-friction daily task — on-call engineers searching for the right
runbook — with the lowest risk, because the agent only reads, never writes. You can
measure the value immediately: time to find the relevant runbook before versus after.
Once that is trusted and used daily, add read-only tool access — query pipeline status,
read error counts, fetch recent run history. Then add write actions with a human
approval step — the agent proposes a fix, a human approves before it executes. Earn
write autonomy gradually. Never start with autonomous write access to production systems.

---

## Quick Reference: Service Selection Cheat Sheet

```
I need to...                                Use...
──────────────────────────────────────────────────────────────────────
Ground LLM responses in my private docs    RAG Engine + RAG Grounding
Search millions of vectors at low latency  Vertex AI Vector Search (BYOI)
Build a custom agent with routing logic    ADK (LlmAgent, SequentialAgent, etc.)
Run multiple agents at the same time       ADK ParallelAgent
Retry until a condition is met             ADK LoopAgent
Keep agent memory across sessions          Memory Bank
Host an agent without managing infra       Agent Engine (Deployments)
Host an agent with GPU or custom network   GKE + ADK
Connect agents across different platforms  A2A Protocol
Publish what my agent can do               Agent Card at /.well-known/agent.json
Connect agent to BigQuery                  BigQuery MCP Server (Cloud Run)
Connect agent to Cloud Logs                Cloud Logging MCP Server (Cloud Run)
Build stateful cyclic workflows            LangGraph (sub-agent within ADK)
Store secrets for A2A OAuth tokens         Secret Manager
Trace multi-agent execution end to end     Cloud Trace
Monitor LLM token costs per agent          Cloud Logging → BigQuery → Looker Studio
Decouple webhook from agent processing     Cloud Pub/Sub
Receive GitHub webhooks for agent trigger  Cloud Run (webhook receiver)
Audit A2A calls for SOC 2 compliance       Cloud Logging sink → GCS bucket (90-day retention)
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
- [Create multi-agent system with ADK and A2A — Google Codelabs](https://codelabs.developers.google.com/codelabs/create-multi-agents-adk-a2a)
