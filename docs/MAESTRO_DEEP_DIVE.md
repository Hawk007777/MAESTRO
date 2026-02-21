# MAESTRO Threat Analyzer — Deep Dive Documentation

> **Purpose of this document**: A comprehensive postmortem and manual reference for understanding exactly what the MAESTRO Threat Analyzer is, how it works internally, and how to apply the MAESTRO threat-modeling framework to any agentic AI system — with or without this tool.

---

## Table of Contents

1. [What Is MAESTRO?](#1-what-is-maestro)
2. [The Seven MAESTRO Layers](#2-the-seven-maestro-layers)
3. [What This Tool Does](#3-what-this-tool-does)
4. [Repository Structure](#4-repository-structure)
5. [Full System Lifecycle](#5-full-system-lifecycle)
6. [Code-Level Data Flow](#6-code-level-data-flow)
7. [AI Flows Deep Dive](#7-ai-flows-deep-dive)
8. [Error Handling and Resilience](#8-error-handling-and-resilience)
9. [How to Apply MAESTRO Manually](#9-how-to-apply-maestro-manually)
10. [Use-Case Walkthrough Example](#10-use-case-walkthrough-example)
11. [Interpreting the Results](#11-interpreting-the-results)
12. [Configuration Reference](#12-configuration-reference)

---

## 1. What Is MAESTRO?

**MAESTRO** stands for **Multi-Agent Environment, Security, Threat, Risk, and Outcome**. It is a threat-modeling *framework* published by the **Cloud Security Alliance (CSA)** specifically designed for **agentic AI systems** — systems where one or more AI agents operate autonomously, delegate tasks to each other, and use external tools.

### Why a New Framework?

Traditional security frameworks (STRIDE, PASTA, OWASP) were designed for deterministic software. Agentic AI introduces fundamentally new attack surfaces that those frameworks do not address:

| Traditional Software | Agentic AI Systems |
|---|---|
| Deterministic behavior | **Non-deterministic** — same input can produce different outputs |
| Fixed identity and roles | **Dynamic identity** — agents acquire and delegate permissions at runtime |
| Explicit trust boundaries | **No trust boundary** — agents freely communicate with other agents |
| Human-initiated actions | **Autonomous actions** — agents act without real-time human approval |
| Single system scope | **Multi-agent interactions** — complex delegation chains and side effects |

MAESTRO addresses these gaps by applying a structured, seven-layer threat analysis that considers both *traditional* threats (which still apply) and *agentic* threats (novel threats introduced by autonomy and multi-agent dynamics).

### The CSA Reference

The framework is fully described in the CSA paper:  
**[Agentic AI Threat Modeling Framework: MAESTRO](https://cloudsecurityalliance.org/blog/2025/02/06/agentic-ai-threat-modeling-framework-maestro)**  
Reading that paper is highly recommended to understand the theoretical underpinnings.

---

## 2. The Seven MAESTRO Layers

MAESTRO decomposes an agentic AI system into seven technology layers, from the lowest-level AI model up to the multi-agent ecosystem. Threats are analyzed per layer.

### Layer 1 — Foundation Models

> *Core AI models: large language models (LLMs), custom-trained AI, embedding models.*

This is the brain of every agent. Threats at this layer affect the fundamental decision-making and knowledge base of any agent in the system.

**What Is Checked:**

| Category | Example Threats |
|---|---|
| Traditional | Model poisoning / backdoor attacks; training data leakage; membership inference (can you tell if a specific data point was in the training set?); model inversion attacks; adversarial inputs that fool the model |
| Agentic | **Non-Determinism**: an agent may respond differently to the same prompt — hard to audit or reproduce security events. **Autonomy**: a poorly aligned model may pursue goals that deviate from intended behavior. **No Trust Boundary**: a model interacting with untrusted agent output may be manipulated via prompt injection. |

**Key Attack: Prompt Injection**  
An attacker embeds malicious instructions inside data the agent reads (e.g., a web page, a document, a tool result), causing the model to follow attacker instructions instead of the system's intended ones.

---

### Layer 2 — Data Operations

> *Data handling: storage, retrieval, processing, vector databases, RAG pipelines, embeddings.*

Agents are only as good as the data they operate on. This layer covers everything that stores or transforms data before the model sees it.

**What Is Checked:**

| Category | Example Threats |
|---|---|
| Traditional | SQL injection into data stores; unauthorized data access; data exfiltration; insecure storage of embeddings; PII leakage from vector databases |
| Agentic | **Non-Determinism**: semantically similar but factually wrong data can be retrieved from a vector DB, leading agents to act on bad information. **Autonomy**: an agent may autonomously write poisoned data into shared storage, affecting other agents. **Agent-to-Agent**: an adversarial agent can poison the shared vector store to manipulate other agents' behavior (indirect prompt injection via RAG). |

**Key Attack: Data Poisoning via RAG**  
An attacker inserts malicious text into a document store. When a retrieval-augmented generation (RAG) agent fetches it as context, the malicious content hijacks the agent's next actions.

---

### Layer 3 — Agent Frameworks

> *Software frameworks and APIs: LangChain, AutoGen, CrewAI, Google ADK, custom orchestration code.*

This layer covers the orchestration code that creates, runs, and manages agents. Bugs or weaknesses here affect all agents the framework controls.

**What Is Checked:**

| Category | Example Threats |
|---|---|
| Traditional | Insecure deserialization; dependency vulnerabilities in framework libraries; insecure default configurations; code injection into tool-calling logic |
| Agentic | **Autonomy**: a misconfigured framework may grant an agent more tool permissions than intended. **Dynamic IAC**: frameworks that assign permissions at runtime can be manipulated to escalate privileges. **Agent-to-Agent**: a compromised orchestration layer can intercept and modify all inter-agent messages. |

**Key Attack: Privilege Escalation via Tool Abuse**  
An agent convinces the framework to call a tool it was not meant to access (e.g., a file system tool instead of a read-only API), because the framework does not validate the agent's authorization before executing the tool.

---

### Layer 4 — Deployment & Infrastructure

> *Servers, containers, cloud resources, networks, APIs hosting agents.*

This layer covers the physical and virtual infrastructure. Compromise here affects the availability and integrity of the whole system.

**What Is Checked:**

| Category | Example Threats |
|---|---|
| Traditional | Container escape; misconfigured cloud IAM; insecure network exposure; DDoS attacks; supply chain attacks on base images |
| Agentic | **Autonomy**: an autonomous agent may provision new cloud resources or open network ports without oversight. **Non-Determinism**: unpredictable agent behavior under load makes capacity planning and security boundary definition harder. **No Trust Boundary**: if agents run without network segmentation, a compromised agent can reach all internal services. |

**Key Attack: Lateral Movement via Agent Autonomy**  
An agent with cloud API access autonomously calls AWS/GCP APIs to exfiltrate data or create back-door accounts, actions that would be flagged for a human but that the agent's access policy allows.

---

### Layer 5 — Evaluation & Observability

> *Monitoring, evaluation, tracing, debugging, logging systems for agent behavior.*

You cannot secure what you cannot see. This layer covers the tools used to observe and evaluate agent behavior.

**What Is Checked:**

| Category | Example Threats |
|---|---|
| Traditional | Log injection (an attacker writes false log entries); insecure logging of sensitive data; incomplete audit trails; alerting system bypass |
| Agentic | **Non-Determinism**: agents produce variable outputs making anomaly detection very difficult. **Autonomy**: autonomous agents take many actions per second — traditional alerting thresholds designed for human-pace actions may miss attacks entirely. **Agent-to-Agent**: complex chains of delegated actions are hard to trace back to the original cause. |

**Key Attack: Log Evasion via Multi-Agent Chains**  
A malicious action is triggered through a long chain of agent delegations (Agent A → Agent B → Agent C → performs harmful action). The action appears in logs as a normal API call and is not attributed to any single suspicious event.

---

### Layer 6 — Security & Compliance

> *Security controls, identity management, access control policies, compliance frameworks (GDPR, HIPAA, SOC2).*

This layer covers the explicit security policies and compliance requirements governing the system.

**What Is Checked:**

| Category | Example Threats |
|---|---|
| Traditional | Broken access control; inadequate encryption; missing input validation; compliance violations (GDPR data retention, HIPAA PHI handling) |
| Agentic | **Dynamic IAC**: agents acquire permissions just-in-time; it is hard to enforce least-privilege when roles are created at runtime. **No Trust Boundary**: agents interacting with external third-party agents may violate data residency or compliance rules without awareness. **Autonomy**: an autonomous agent may take an action that violates a compliance rule (e.g., storing PII in a non-compliant region) without any human review gate. |

**Key Attack: Compliance Bypass via Agentic Delegation**  
A compliant agent A delegates to less-restricted agent B, which then processes regulated data outside the compliance boundary. Agent A technically followed the rules; agent B had no compliance controls.

---

### Layer 7 — Agent Ecosystem

> *The broader multi-agent environment: agent marketplaces, third-party agents, external APIs, agent-to-agent protocols (A2A, MCP).*

This is the outermost layer — the entire environment of agents and services the system operates within. This layer is unique to agentic systems and has no equivalent in traditional threat modeling.

**What Is Checked:**

| Category | Example Threats |
|---|---|
| Traditional | Third-party integration risks; vendor lock-in; API versioning vulnerabilities; supply chain risks from external agent registries |
| Agentic | **No Trust Boundary**: your agents interact with untrusted third-party agents discovered dynamically; there is no pre-established trust. **Autonomy**: agents may discover and invoke new external agents without human review. **Agent-to-Agent**: a malicious agent in the ecosystem can impersonate a legitimate one (agent spoofing). **Dynamic IAC**: permissions granted to one external agent can cascade through that agent's own delegation chain. |

**Key Attack: Agent Impersonation (A2A Spoofing)**  
An attacker deploys a malicious agent with an Agent Card that mimics a legitimate service. Your system's agent discovers it via capability lookup and delegates a sensitive task to it, exposing data to the attacker.

---

## 3. What This Tool Does

The MAESTRO Threat Analyzer is a **web application** that automates the MAESTRO threat-modeling process using AI:

1. **You describe your system** — paste an architecture description or pick a preset.
2. **The tool iterates through all 7 layers** — for each layer, it calls an LLM with a specialized prompt.
3. **The LLM identifies threats** — both traditional and agentic threats for that layer, given your architecture.
4. **The LLM recommends mitigations** — concrete recommendations, with reasoning and caveats.
5. **The tool generates a report** — executive summary, architecture diagram (Mermaid), downloadable PDF.

> **Important**: This tool provides AI-assisted suggestions. It is not a substitute for a human security expert. Always review the output critically and supplement it with manual analysis.

---

## 4. Repository Structure

```
MAESTRO/
├── src/
│   ├── ai/                          # Genkit AI backend
│   │   ├── genkit.ts                # LLM provider initialization (Google/OpenAI/Ollama)
│   │   ├── dev.ts                   # Genkit dev-server entry point
│   │   └── flows/                   # Individual AI workflows
│   │       ├── suggest-threats-for-layer.ts  # Threat analysis per layer
│   │       ├── recommend-mitigations.ts      # Mitigation recommendations
│   │       ├── generate-executive-summary.ts # High-level summary
│   │       └── generate-architecture-diagram.ts # Mermaid diagram
│   │
│   ├── app/                         # Next.js App Router
│   │   ├── page.tsx                 # Main UI component (orchestrates everything)
│   │   ├── actions.ts               # Server actions (bridge between UI and AI flows)
│   │   └── layout.tsx               # Root layout
│   │
│   ├── components/                  # React components
│   │   ├── sidebar-input-form.tsx   # Architecture input + use-case selector
│   │   ├── layer-card.tsx           # Displays one MAESTRO layer's results
│   │   ├── mermaid-diagram.tsx      # Renders Mermaid diagrams
│   │   ├── error-boundary.tsx       # Error boundaries for graceful failures
│   │   ├── icons.tsx                # Custom icons (spinner)
│   │   └── ui/                      # shadcn/ui component library
│   │
│   ├── data/                        # Static reference data
│   │   ├── maestro.ts               # The 7 MAESTRO layer definitions
│   │   └── use-cases.ts             # 10 pre-populated architecture descriptions
│   │
│   └── lib/                         # Utility and infrastructure code
│       ├── types.ts                 # TypeScript interfaces (LayerData, Mitigation, etc.)
│       ├── errors.ts                # Error codes, severity, MaestroError class
│       ├── ai-error-handler.ts      # Classifies Genkit errors for retry decisions
│       ├── retry-utils.ts           # Exponential backoff retry mechanism
│       └── utils.ts                 # Class name merging utility
│
├── docs/
│   ├── blueprint.md                 # Original feature spec / design blueprint
│   └── MAESTRO_DEEP_DIVE.md         # This document
│
├── package.json                     # Dependencies and scripts
├── next.config.ts                   # Next.js configuration
├── tailwind.config.ts               # Tailwind CSS configuration
├── tsconfig.json                    # TypeScript configuration
├── vitest.config.ts                 # Test framework configuration
├── Dockerfile                       # Container deployment
├── apphosting.yaml                  # Firebase App Hosting configuration
└── README.md                        # Quick-start guide
```

---

## 5. Full System Lifecycle

This section walks through everything that happens from the moment you open the browser to the moment you download a PDF report.

### Phase 0 — Application Startup

Two processes must run concurrently:

```
Terminal 1: npm run dev
  → Starts Next.js on http://localhost:9002
  → Serves the React UI
  → Runs server actions (actions.ts) via Next.js server components

Terminal 2: npm run genkit:watch  (or genkit:dev)
  → Starts the Genkit AI flow runner
  → Loads all flow definitions from src/ai/flows/
  → Connects to the configured LLM provider (Google/OpenAI/Ollama)
  → Watches for code changes and auto-reloads
```

**Why two processes?**  
Next.js handles the web layer (routing, server-side rendering, server actions). Genkit handles the AI orchestration layer (flow execution, prompt management, LLM API calls). They work together: a Next.js server action calls a Genkit flow function, which in turn calls the LLM API.

### Phase 1 — User Input

The user interacts with the `SidebarInputForm` component:

1. **Option A — Select a preset**: Picks one of 10 pre-defined use cases from a dropdown (e.g., "Medical Diagnosis Assistant"). The corresponding architecture description from `src/data/use-cases.ts` is loaded into the textarea automatically.

2. **Option B — Write a custom description**: Types a free-form description of their system architecture (minimum 50 characters, maximum 5000). Zod schema validation (`react-hook-form` + `zod`) runs on submit to check the length constraints.

3. **Clicks "Generate Analysis"**: The form calls `handleAnalyze()` in `page.tsx`.

### Phase 2 — Layer-by-Layer Analysis

`page.tsx` iterates through the 7 layers in `MAESTRO_LAYERS` (from `src/data/maestro.ts`) **sequentially** (not in parallel, to avoid rate limiting and to allow real-time progress display):

```
For each of the 7 layers:
  1. Set layer status → "analyzing"
  2. Log: "Analyzing [Layer Name]..."
  3. Call suggestThreat(architectureDescription, layerName, layerDescription)
     → Goes to actions.ts → suggestThreatsForLayer Genkit flow → LLM
     → Returns: threatAnalysis (Markdown string)
  4. Store threat in layer state
  5. Log: "Threats identified for [Layer Name]. Generating mitigations..."
  6. Call recommendMitigation(threatAnalysis, layerName)
     → Goes to actions.ts → recommendMitigations Genkit flow → LLM
     → Returns: { recommendation, reasoning, caveats }
  7. Store mitigation in layer state
  8. Set layer status → "complete"
  9. Log: "[Layer Name] analysis complete."
```

If the user clicks "Stop Analysis" at any point, an `AbortController`-style flag (`analysisRef.current.shouldStop`) is checked before each layer, stopping the loop gracefully.

### Phase 3 — Post-Analysis

After all 7 layers complete:

1. **Executive Summary**: `getExecutiveSummary(architectureDescription, allLayerData)` is called. All 7 layers' threat and mitigation data is bundled and sent to the LLM for a high-level summary aimed at a leadership audience.

2. **Architecture Diagram**: When the user clicks "Generate Diagram", `getArchitectureDiagram(architectureDescription)` is called. The LLM generates Mermaid.js syntax representing the system. The `MermaidDiagram` component renders it visually in the browser.

### Phase 4 — PDF Export

The user clicks "Export PDF". The app uses `jsPDF` to:

1. Render the Mermaid diagram as an SVG and embed it.
2. Add the executive summary.
3. For each of the 7 layers, add the layer name, threat analysis text, and mitigation recommendation.
4. Download the file as `maestro-analysis-[timestamp].pdf`.

---

## 6. Code-Level Data Flow

The following traces a single layer analysis call through every function and file:

```
page.tsx
  handleAnalyze()
    │
    ├─ suggestThreat(architectureDescription, layerName, layerDescription)
    │    [actions.ts - "use server"]
    │    │
    │    └─ withRetry(() => {
    │         suggestThreatsForLayer({ architectureDescription, layerName, layerDescription })
    │         [src/ai/flows/suggest-threats-for-layer.ts]
    │         │
    │         └─ suggestThreatsForLayerFlow(input)
    │              │
    │              └─ prompt(input)   ← Genkit definePrompt()
    │                   │
    │                   └─ ai.generate(prompt_text_with_handlebars)
    │                        │
    │                        └─ LLM API call (Google Gemini / OpenAI / Ollama)
    │                             │
    │                             └─ Returns: { threatAnalysis: "## Traditional Threats\n..." }
    │         })
    │
    └─ recommendMitigation(threatAnalysis, layerName)
         [actions.ts - "use server"]
         │
         └─ withRetry(() => {
              recommendMitigations({ threatDescription: threatAnalysis, layer: layerName })
              [src/ai/flows/recommend-mitigations.ts]
              │
              └─ recommendMitigationsFlow(input)
                   │
                   └─ prompt(input)   ← Genkit definePrompt()
                        │
                        └─ LLM API call
                             │
                             └─ Returns: {
                                  recommendation: "Implement input validation...",
                                  reasoning: "Because prompt injection...",
                                  caveats: "This approach may not cover..."
                                }
              })
```

**Server Actions as the Bridge**  
`actions.ts` uses Next.js `"use server"` directive, meaning these functions run on the server — never in the browser. This keeps LLM API keys secure (never sent to the client) and allows the UI to call them like regular async functions.

**Genkit as the AI Orchestration Layer**  
Genkit wraps the raw LLM calls with:
- Schema validation (Zod) on both input and output
- Prompt templating (Handlebars syntax: `{{{variable}}}`)
- Named flow definitions (for traceability in the Genkit dev UI)
- Output parsing (structured JSON from the LLM response)

---

## 7. AI Flows Deep Dive

### Flow 1: `suggestThreatsForLayer`

**File**: `src/ai/flows/suggest-threats-for-layer.ts`

**Input Schema**:
```typescript
{
  architectureDescription: string  // The full system description
  layerName: string                 // e.g., "Foundation Models"
  layerDescription: string          // e.g., "Core AI models..."
}
```

**Output Schema**:
```typescript
{
  threatAnalysis: string  // Markdown-formatted threat analysis
}
```

**What the Prompt Instructs the LLM to Do**:
1. Act as a security analyst specializing in multi-agent systems.
2. Read the system architecture description and the specific layer being analyzed.
3. Consider these five agentic factors: Non-Determinism, Autonomy, No Trust Boundary, Dynamic Identity and Access Control, Agent-to-Agent interactions.
4. Produce a Markdown report split into **Category 1: Traditional Threats** and **Category 2: Agentic Threats**.
5. For each agentic factor, either describe the resulting threat or state it does not apply.

**Sample Output Structure**:
```markdown
## Traditional Threats

### Model Poisoning
Training data for the foundation model could be...

### Prompt Injection
An attacker could embed malicious instructions...

## Agentic Threats

### Non-Determinism
The model's non-deterministic nature means...

### No Trust Boundary
Since agents communicate directly with each other...
```

---

### Flow 2: `recommendMitigations`

**File**: `src/ai/flows/recommend-mitigations.ts`

**Input Schema**:
```typescript
{
  threatDescription: string  // The full threat analysis from Flow 1
  layer: string              // The MAESTRO layer name
}
```

**Output Schema** (structured JSON):
```typescript
{
  recommendation: string  // The main mitigation action
  reasoning: string       // Why this mitigation works
  caveats: string         // Limitations or edge cases
}
```

**What the Prompt Instructs the LLM to Do**:  
Act as a cybersecurity expert. Given the threats identified in a specific layer, recommend concrete mitigations with reasoning and limitations. The structured output (vs. free-form Markdown) makes it easy for the UI to display each piece distinctly with appropriate icons.

---

### Flow 3: `generateExecutiveSummary`

**File**: `src/ai/flows/generate-executive-summary.ts`

**Input Schema**:
```typescript
{
  architectureDescription: string
  analysisResults: LayerData[]  // All 7 layers' threat + mitigation data
}
```

**Output Schema**:
```typescript
{
  summary: string  // Markdown executive summary
}
```

**What the Prompt Instructs the LLM to Do**:
1. Briefly acknowledge the analyzed architecture.
2. Highlight the most critical threats across all layers.
3. Summarize the key mitigation themes.
4. Conclude with a defense-in-depth statement.
5. Target a leadership audience (concise, strategic).
6. Include a link to the MAESTRO framework paper.

The prompt uses Handlebars `{{#each}}` templating to loop over all layer results and inject them into the LLM context.

---

### Flow 4: `generateArchitectureDiagram`

**File**: `src/ai/flows/generate-architecture-diagram.ts`

**Input Schema**:
```typescript
{
  architectureDescription: string
}
```

**Output Schema**:
```typescript
{
  mermaidCode: string  // Mermaid graph TD syntax
}
```

**What the Prompt Instructs the LLM to Do**:
1. Generate a `graph TD` (top-down) Mermaid diagram.
2. Use simple node IDs and labels (`A[Agent Name]`).
3. Use `-->` connectors for relationships.
4. **Critically, avoid parentheses, brackets, or special characters** in node labels — these break Mermaid's parser.
5. Return *only* the Mermaid code in a fenced code block.

**Post-Processing**:  
The flow strips the `\`\`\`mermaid` and `\`\`\`` fences from the output before storing the raw Mermaid syntax. The `MermaidDiagram` React component then passes this to the Mermaid.js library for browser-side rendering to SVG.

---

## 8. Error Handling and Resilience

### Error Classification Hierarchy

```
ErrorCode (enum)
  AI_SERVICE_UNAVAILABLE   → severity: HIGH   → retryable: yes
  AI_RATE_LIMIT_EXCEEDED   → severity: MEDIUM → retryable: yes (with delay)
  AI_INVALID_RESPONSE      → severity: MEDIUM → retryable: yes
  AI_TIMEOUT               → severity: MEDIUM → retryable: yes
  NETWORK_ERROR            → severity: MEDIUM → retryable: yes
  LAYER_PROCESSING_FAILED  → severity: HIGH   → retryable: depends
  ANALYSIS_INVALID_INPUT   → severity: LOW    → retryable: no
  PDF_GENERATION_FAILED    → severity: MEDIUM → retryable: yes
  UNKNOWN_ERROR            → severity: MEDIUM → retryable: yes
```

### Retry Strategy (Exponential Backoff)

Every server action is wrapped in `withRetry()`:

```
Attempt 1: immediate
  → Failure? Wait 1s + random jitter (0–1s)
Attempt 2: after ~1–2s
  → Failure? Wait 2s + random jitter
Attempt 3 (final): after ~2–3s
  → Failure? Throw the error to the UI
```

**Configuration** (`DEFAULT_RETRY_CONFIG`):
- `maxAttempts: 3`
- `baseDelay: 1000ms`
- `maxDelay: 10000ms`
- `backoffMultiplier: 2`
- Jitter: adds up to 1000ms random to prevent thundering herd

### Error Classification (AIErrorHandler)

`AIErrorHandler.classifyGenkitError()` inspects the error message to determine the error code:

| Error message pattern | Classified as |
|---|---|
| "rate limit", "quota" | `AI_RATE_LIMIT_EXCEEDED` |
| "unavailable", "503", "502" | `AI_SERVICE_UNAVAILABLE` |
| "timeout", "timed out" | `AI_TIMEOUT` |
| "unauthorized", "401", "api key" | `AI_SERVICE_UNAVAILABLE` |
| "parse", "json", "format" | `AI_INVALID_RESPONSE` |
| "network", "connection", "fetch" | `NETWORK_ERROR` |
| anything else | `UNKNOWN_ERROR` |

---

## 9. How to Apply MAESTRO Manually

This section is a **complete guide** for performing a MAESTRO threat-modeling exercise by hand, without using this tool. This is valuable for:
- Situations where you cannot use AI/API services (air-gapped environments, compliance restrictions)
- Supplementing and validating the AI tool's output
- Learning to deeply internalize the framework

### Step 0: Prepare Your System Description

Write a clear, structured architecture description covering:

- **Components**: What agents exist? What services/APIs do they use?
- **Protocols**: How do agents communicate? (A2A, MCP, REST, gRPC, message queues?)
- **Data flows**: What data moves where? What is sensitive?
- **Autonomy level**: Can agents act autonomously? On what timescales?
- **External integrations**: What third-party agents, services, or tools does the system connect to?
- **Human oversight**: At what points does a human review or approve actions?

**Example (abbreviated)**:
```
A medical AI assistant where:
- A PatientData agent fetches patient records from an EHR API (MCP)
- A SymptomAnalysis agent receives data via A2A and cross-references 
  a medical knowledge database (MCP)
- A Reporting agent compiles findings for the doctor via A2A
- All data is PHI (HIPAA regulated)
- No human reviews individual agent-to-agent messages
```

---

### Step 1: Analyze Layer 1 — Foundation Models

**Ask yourself:**

**Traditional Threats:**
- Could training data for the LLM contain poisoned, biased, or private information?
- Can an attacker extract training data through repeated queries (model inversion)?
- Can someone determine if a specific record was in the training set (membership inference)?
- Is the model susceptible to adversarial inputs that degrade performance?

**Agentic Threats:**
- *Non-Determinism*: If the model outputs vary unpredictably, could this lead to inconsistent security decisions? Can you reproduce a security incident?
- *Autonomy*: If the model is misaligned, could it autonomously pursue unintended goals? What are the blast radius limits?
- *No Trust Boundary*: What external content does the model read? Could an attacker embed instructions in documents, emails, or tool results that the model would follow (prompt injection)?
- *Dynamic IAC*: Does the model receive different system prompts from different agents? Could one agent convince it to act outside its intended role?
- *Agent-to-Agent*: Could messages from other agents contain adversarial content designed to manipulate this model?

**Fill in this table:**

| Threat | Category | Likelihood (L/M/H) | Impact (L/M/H) | Risk = L×I |
|---|---|---|---|---|
| Prompt injection via document RAG | Agentic | H | H | Critical |
| Model inversion of patient data | Traditional | L | H | Medium |
| ... | ... | ... | ... | ... |

---

### Step 2: Analyze Layer 2 — Data Operations

**Ask yourself:**

**Traditional Threats:**
- Are databases and vector stores properly access-controlled?
- Is data encrypted at rest and in transit?
- Could an attacker inject malicious data into the training pipeline or knowledge base?
- Is PII or regulated data (PHI, PCI-DSS) handled according to regulations?
- What is the data retention and deletion policy?

**Agentic Threats:**
- *Non-Determinism*: Could semantic search in a vector DB return subtly incorrect but plausible results that lead agents astray?
- *Autonomy*: Could an autonomous agent write data to shared storage that other agents will consume (data poisoning via agentic write)?
- *Agent-to-Agent*: Can one agent observe or modify another agent's data cache or working memory?
- *No Trust Boundary*: Is there any isolation between different agents' data spaces?

---

### Step 3: Analyze Layer 3 — Agent Frameworks

**Ask yourself:**

**Traditional Threats:**
- Are framework dependencies (libraries, packages) kept updated and scanned for CVEs?
- Is the framework's code properly validated against injection attacks?
- Are framework API keys or service accounts properly secured?
- Is serialization/deserialization of agent state secured?

**Agentic Threats:**
- *Autonomy*: Does the framework allow an agent to call any tool, or are there per-agent allowlists?
- *Dynamic IAC*: When the framework assigns a tool permission to an agent at runtime, is there any authorization check?
- *Agent-to-Agent*: Does the framework validate the identity of agents that submit tasks? Can a rogue agent inject itself into the workflow?
- *No Trust Boundary*: If one agent in the framework is compromised, can it read/modify other agents' contexts?

---

### Step 4: Analyze Layer 4 — Deployment & Infrastructure

**Ask yourself:**

**Traditional Threats:**
- Are containers hardened? (non-root user, minimal base images, read-only filesystems)
- Are cloud IAM permissions following least-privilege?
- Is the network segmented appropriately?
- Are agent API endpoints exposed to the internet when they shouldn't be?
- Is there DDoS protection?

**Agentic Threats:**
- *Autonomy*: Can agents autonomously provision cloud resources, open ports, or create IAM roles? Who audits this?
- *Non-Determinism*: If agent behavior is unpredictable under load, how does the infrastructure respond to unexpected API call volumes?
- *No Trust Boundary*: Are agents running in the same network segment as sensitive internal services? If one agent is compromised, can it reach the database directly?

---

### Step 5: Analyze Layer 5 — Evaluation & Observability

**Ask yourself:**

**Traditional Threats:**
- Are logs protected from tampering?
- Do logs capture enough detail for forensic investigation?
- Is sensitive data being logged unintentionally (PII, API keys in logs)?
- Are monitoring alerts properly configured and tuned?

**Agentic Threats:**
- *Non-Determinism*: How do you detect anomalous agent behavior when "normal" behavior is already variable? What baselines have you established?
- *Autonomy*: Autonomous agents act at machine speed. Can your alerting system keep up? Are alert thresholds appropriate for agent-pace actions?
- *Agent-to-Agent*: In a multi-agent chain, can you trace an action back to its root cause? Is there end-to-end correlation of agent call spans?
- *No Trust Boundary*: If an external agent is involved in an incident, do your logs capture its identity and actions?

---

### Step 6: Analyze Layer 6 — Security & Compliance

**Ask yourself:**

**Traditional Threats:**
- What compliance frameworks apply? (GDPR, HIPAA, SOC2, PCI-DSS, NIST AI RMF)
- Are all data handling practices compliant?
- Is there a formal access control policy? Is it enforced technically?
- Are encryption standards current?

**Agentic Threats:**
- *Dynamic IAC*: How do you enforce compliance when agent permissions are assigned at runtime? Is there a policy engine that evaluates every tool call against compliance rules?
- *No Trust Boundary*: When your agents interact with external agents, do those external agents' data handling practices comply with your regulations? How do you verify?
- *Autonomy*: Could an autonomous agent take a non-compliant action (e.g., log sensitive data, store data in an unapproved region) without a human review gate?
- *Agent-to-Agent*: If a compliance audit requires showing who decided to perform a sensitive action, can you attribute it to a specific agent decision and not just "the system"?

---

### Step 7: Analyze Layer 7 — Agent Ecosystem

**Ask yourself:**

**Traditional Threats:**
- How do you vet third-party agents before integrating them?
- What SLA and security standards do external agent providers meet?
- Is there a process for revoking access to a third-party agent?
- Are there contractual obligations (DPA, SLA) with external agent providers?

**Agentic Threats:**
- *No Trust Boundary*: How does your system verify the identity of an agent it discovers dynamically? (Agent Card / A2A capability discovery is unauthenticated by default)
- *Autonomy*: Could your agent autonomously discover and connect to a new third-party agent it has never used before, without human approval?
- *Agent-to-Agent*: Could a malicious actor publish a fake agent to a public registry that mimics a legitimate service? How would your agent distinguish it?
- *Dynamic IAC*: If you grant an external agent a capability, what is the scope of that grant? Can the external agent further delegate it to other agents in its own ecosystem?

---

### Step 8: Prioritize and Document

For each threat identified, complete the following:

```
THREAT RECORD

ID: T-L1-001                          # Layer number and sequence
Layer: Foundation Models
Threat Name: Prompt Injection via RAG
Category: Agentic — No Trust Boundary
MAESTRO Factor: No Trust Boundary

Description:
  An attacker embeds malicious instructions in documents stored in the 
  vector database. When the PatientData agent retrieves these documents 
  as context, the LLM follows attacker instructions instead of the 
  intended system behavior.

Likelihood: High
  Rationale: Documents from external sources are ingested without validation.

Impact: Critical
  Rationale: Could cause the LLM to exfiltrate PHI or recommend incorrect 
  treatments.

Risk Level: Critical

Mitigation:
  1. Implement strict input sanitization on all RAG-retrieved documents.
  2. Use a separate "sanitization agent" to validate retrieved content 
     before passing to the main agent.
  3. Apply output validation to check that agent responses conform to 
     expected patterns.

Reasoning:
  Layered validation prevents a single point of injection and reduces 
  blast radius if one layer fails.

Caveats:
  Perfect sanitization is impossible; sophisticated prompt injection 
  may evade detection. Defense-in-depth is essential.

Residual Risk: Medium (after mitigation)
Owner: Security Team
Review Date: [quarterly]
```

---

### Step 9: Create the Threat Matrix

Compile all threats into a single matrix:

| ID | Layer | Threat | Category | Likelihood | Impact | Risk | Mitigation Status |
|---|---|---|---|---|---|---|---|
| T-L1-001 | Foundation Models | Prompt Injection via RAG | Agentic | H | C | Critical | In Progress |
| T-L1-002 | Foundation Models | Model Inversion | Traditional | L | H | Medium | Accepted |
| T-L2-001 | Data Operations | Data Poisoning via Agentic Write | Agentic | M | H | High | Planned |
| T-L3-001 | Agent Frameworks | Privilege Escalation via Tool Abuse | Agentic | M | C | Critical | Planned |
| ... | ... | ... | ... | ... | ... | ... | ... |

---

### Step 10: Write the Executive Summary

Structure it as follows:

```markdown
# MAESTRO Threat Analysis — Executive Summary
## System: [System Name]
## Date: [Date]
## Analyst: [Name]

### Overview
[2-3 sentences describing the system and scope of analysis]

### Critical Findings
- [Most severe threat 1 with layer reference]
- [Most severe threat 2 with layer reference]

### Key Themes
- [Recurring theme across multiple layers, e.g., "Lack of agent identity verification"]

### Priority Mitigations (Top 5)
1. [Highest priority action] — addresses threats T-L1-001, T-L3-001
2. [Second priority] — addresses threats T-L2-001
...

### Defense-in-Depth Recommendation
[Statement about the need for overlapping controls across all layers]

### Reference
MAESTRO Framework: https://cloudsecurityalliance.org/blog/2025/02/06/agentic-ai-threat-modeling-framework-maestro
```

---

## 10. Use-Case Walkthrough Example

Let's trace through the **Medical Diagnosis Assistant** use case to see how both the tool and a manual MAESTRO exercise would produce results.

### The Architecture

```
PatientData Agent
  → fetches from EHR API via MCP (FHIR data, historical records)
  → sends to SymptomAnalysis Agent via A2A

SymptomAnalysis Agent  
  → receives patient data via A2A
  → cross-references MedicalKnowledgeDB via MCP
  → sends findings to Reporting Agent via A2A

Reporting Agent
  → aggregates from other agents via A2A
  → compiles summary for doctor

All data = PHI (HIPAA regulated)
No human reviews individual agent messages
```

### Sample MAESTRO Analysis

**Layer 1 — Foundation Models**

*Traditional*: The LLMs used by each agent may have been trained on data containing medical misinformation or biased clinical data, leading to incorrect differential diagnoses.

*Agentic — No Trust Boundary*: The SymptomAnalysis agent receives raw patient data from PatientData agent via A2A without cryptographic verification. A compromised PatientData agent or a man-in-the-middle could inject false patient data, causing the LLM to analyze fabricated symptoms.

**Layer 2 — Data Operations**

*Traditional*: The MedicalKnowledgeDB may contain outdated treatment information if it is not regularly updated and audited.

*Agentic — Autonomy*: The PatientData agent autonomously fetches and caches PHI from the EHR. If cache invalidation is not properly managed, stale patient data could persist and be analyzed for incorrect diagnoses.

**Layer 6 — Security & Compliance**

*Agentic — Dynamic IAC*: HIPAA requires that access to PHI be logged with the requester's identity. If the SymptomAnalysis agent accesses patient data via A2A without explicit attribution (since the A2A message may not carry the requesting physician's identity), the audit trail may be insufficient for HIPAA compliance.

*Agentic — No Trust Boundary*: If the Reporting agent sends its summary to an external system (e.g., a third-party PDF generator), PHI may leave the HIPAA-compliant boundary without proper data processing agreements.

**Layer 7 — Agent Ecosystem**

*Agentic — Agent Spoofing*: The SymptomAnalysis agent discovers the MedicalKnowledgeDB MCP server via capability lookup. A malicious party could deploy a fake MCP server that returns dangerous drug interaction data or incorrect diagnosis suggestions if the agent does not verify the server's identity.

---

## 11. Interpreting the Results

### Reading a Layer Card

Each `LayerCard` in the UI shows:

- **Status badge**: `Pending` (grey) → `Analyzing` (yellow, spinning) → `Complete` (green) or `Error` (red)
- **Threats section** (`ShieldAlert` icon): The AI's Markdown-formatted threat analysis, split into Traditional and Agentic categories. Read this critically.
- **Mitigation section** (`ShieldCheck` icon): The main recommended action.
- **Reasoning section** (`Lightbulb` icon): Why the mitigation works. Useful for understanding and customizing the approach.
- **Caveats section** (`AlertTriangle` icon): Limitations of the mitigation. **Always read the caveats** — they indicate where the AI is uncertain or where the mitigation is incomplete.

### Critical Evaluation Checklist

When reviewing AI-generated MAESTRO output, ask:

- [ ] Does the threat actually apply to my system as described, or is it generic?
- [ ] Is the likelihood assessment appropriate given my specific environment?
- [ ] Is the recommended mitigation technically feasible in my stack?
- [ ] Does the mitigation introduce new threats? (e.g., a centralized policy engine becomes a new single point of failure)
- [ ] Are there threats that the AI missed, especially domain-specific ones?
- [ ] Have I reviewed the caveats section and addressed any gaps?
- [ ] Are the most critical threats prioritized for immediate action?

### Common AI Blind Spots

The AI tool is strong at identifying *categories* of threats but may miss:
- **Deployment-specific details**: Threats unique to your exact cloud provider, region, or configuration
- **Business logic threats**: Attacks that exploit the specific business domain rather than generic security properties
- **Implicit trust assumptions**: Relationships not explicitly mentioned in your architecture description
- **Regulatory nuances**: Country-specific or industry-specific compliance requirements beyond the common frameworks

Always supplement AI output with domain-expert review.

---

## 12. Configuration Reference

### Environment Variables

Create a `.env` file in the project root:

```bash
# Required: LLM Provider (defaults to "google" if not set)
LLM_PROVIDER=google   # or: openai, ollama

# Google Gemini (when LLM_PROVIDER=google)
GEMINI_API_KEY=your_api_key_here

# OpenAI (when LLM_PROVIDER=openai)
OPENAI_API_KEY=your_api_key_here

# Ollama (when LLM_PROVIDER=ollama) — local model server
OLLAMA_SERVER_ADDRESS=http://localhost:11434

# Optional: Override the default model for any provider
LLM_MODEL=model-name
```

### Default Models by Provider

| Provider | Default Model | Notes |
|---|---|---|
| Google (`google`) | `gemini-2.5-flash` | Fast, cost-effective, good reasoning |
| OpenAI (`openai`) | `gpt-4o-mini` | Balanced cost/quality |
| Ollama (`ollama`) | `qwen3:8b` | Runs locally, no API costs, slower |

### Development Commands

```bash
npm run dev           # Next.js frontend on port 9002 (Turbopack)
npm run genkit:dev    # Genkit AI flows backend (standard)
npm run genkit:watch  # Genkit backend with hot-reload (recommended for development)
npm run build         # Production build
npm run start         # Run production server
npm run lint          # ESLint
npm run typecheck     # TypeScript type checking
npm run test          # Run tests in watch mode (Vitest)
npm run test:run      # Run tests once
npm run test:coverage # Run tests with coverage report
```

### Docker Deployment

A `Dockerfile` is included for containerized deployment. The `apphosting.yaml` supports **Firebase App Hosting** as a managed deployment option.

---

## Appendix: MAESTRO Quick Reference Card

| Layer | Focus Area | Key Traditional Threats | Key Agentic Threats |
|---|---|---|---|
| 1. Foundation Models | LLMs, embeddings, model weights | Poisoning, inversion, adversarial inputs | Prompt injection, goal misalignment, non-deterministic exploits |
| 2. Data Operations | Databases, vector stores, RAG | Injection, unauthorized access, PII leakage | Data poisoning via agentic writes, semantic search manipulation |
| 3. Agent Frameworks | Orchestration code, APIs | Dependency CVEs, insecure defaults | Privilege escalation, tool abuse, framework compromise |
| 4. Deployment & Infrastructure | Cloud, containers, networks | Container escape, misconfigured IAM, DDoS | Autonomous resource provisioning, lateral movement |
| 5. Evaluation & Observability | Monitoring, logging, tracing | Log injection, data in logs, alert bypass | Anomaly detection gaps, multi-agent trace evasion |
| 6. Security & Compliance | Policies, RBAC, regulations | Broken access control, compliance gaps | Dynamic permission escalation, compliance bypass via delegation |
| 7. Agent Ecosystem | Third-party agents, marketplaces | Third-party risk, supply chain | Agent spoofing, unchecked external delegation, ecosystem poisoning |

---

*This documentation was created to provide a comprehensive understanding of the MAESTRO Threat Analyzer and the MAESTRO framework for anyone looking to understand, use, or extend this tool — and to apply the framework manually to their own agentic AI systems.*

*Reference: [Agentic AI Threat Modeling Framework: MAESTRO — Cloud Security Alliance](https://cloudsecurityalliance.org/blog/2025/02/06/agentic-ai-threat-modeling-framework-maestro)*
