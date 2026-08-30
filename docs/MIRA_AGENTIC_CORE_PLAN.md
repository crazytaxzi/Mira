# MIRA 2.0 — Agentic Core Architecture and Delivery Plan

**Status:** Architecture baseline  
**Repository:** `crazytaxzi/Mira`  
**Default model:** `gpt-5.6-luna`  
**Primary runtime:** Python 3.12 on Windows  
**Scope:** Build the reasoning, tool, memory, action, and orchestration core around the existing STT and TTS systems.

## 1. Product Goal

MIRA must be a persistent, voice-accessible agent that can:

- understand a request;
- decide whether it can answer directly or needs tools;
- select and call the correct tools;
- inspect tool results and continue working;
- remember relevant context across sessions;
- take real actions under explicit, durable permission rules;
- recover from tool errors without silently abandoning the task;
- explain the result and provide an action receipt;
- remain one coherent MIRA rather than a pile of unrelated bots.

The target is not merely a chatbot with function buttons. The target is a controlled agent loop:

`input -> context -> model decision -> tool/policy gate -> action -> observation -> next decision -> final response -> memory/audit`

## 2. Current Platform Decision

Build on the **OpenAI Responses API** and the **OpenAI Agents SDK**, not the retired Assistants API.

`gpt-5.6-luna` is the default model. It supports streaming, function calling, structured outputs, web search, file search, image generation, code interpreter, hosted shell, apply patch, skills, computer use, MCP, and tool search through the Responses API. It does not accept or emit audio directly, which is fine because MIRA already has separate STT and TTS components.

The implementation must use the model ID through configuration rather than hard-coding it throughout the codebase:

```env
MIRA_MODEL=gpt-5.6-luna
MIRA_REASONING_EFFORT=medium
```

Reasoning effort policy:

| Task class | Default effort |
|---|---:|
| simple conversation or one read-only lookup | low |
| ordinary tool use and multi-step requests | medium |
| complex diagnosis, planning, or recovery | high |
| explicitly requested deep work | xhigh or max |

Luna remains MIRA's default brain. Model escalation is not part of the first milestone. A later optional router may permit a deliberately scoped escalation to Terra or Sol, but only after Luna's behavior is measured.

## 3. Critical Reality Check: “All the Same Tools”

MIRA can receive **functional equivalents** of the tools available in ChatGPT, but those tools are not magically inherited from a ChatGPT account or this repository. Every capability must be provided through one of four routes:

1. **OpenAI-hosted tools** in the Responses API.
2. **Application-owned function tools** implemented in MIRA.
3. **Remote MCP servers or OpenAI connectors** with the appropriate authorization.
4. **A local tool gateway** that safely exposes selected machine and service actions.

Credentials, OAuth grants, local file access, Discord access, GitHub access, Google access, Minecraft administration, browser control, and operating-system control each require a concrete adapter and an authorization policy. “Agentic” does not mean “unrestricted administrator.” Unrestricted tools produce an impressive demo right before they produce an expensive mess.

## 4. Capability Map

| Desired capability | MIRA implementation |
|---|---|
| current web research with citations | Responses API `web_search` |
| search uploaded/private reference material | `file_search` plus vector stores |
| calculations and data analysis | `code_interpreter` and small deterministic local functions |
| inspect images/screenshots | GPT-5.6 image input |
| generate images | Responses API image generation tool |
| use a browser or visual UI | computer-use harness in an isolated browser/container |
| run commands | local shell tool restricted by workspace, executable, and risk policy |
| edit code | repository-scoped file tools and apply-patch workflow |
| GitHub issues, files, commits, PRs, Actions | GitHub API or GitHub MCP adapter with scoped credentials |
| Gmail, Calendar, Contacts, Drive | OpenAI connector/MCP or direct Google APIs with OAuth |
| Discord, Twitch, YouTube/music, OBS | application-owned adapters or private MCP tools |
| Minecraft administration | existing/private server administration tool exposed through the local gateway |
| reminders and recurring jobs | durable local job store plus event scheduler |
| personal context and long-term memory | MIRA-owned semantic and episodic memory service |
| local notifications | Windows notification adapter |
| location, weather, time | deterministic local/API tools; never guessed from model memory |

## 5. Architecture

### 5.1 Main components

1. **Voice bridge**
   - Receives final STT utterances.
   - Sends streamed, speakable output to TTS.
   - Supports barge-in, cancellation, and mute/stop events.
   - Does not expose raw tool output to TTS.

2. **Agent runtime**
   - Runs the MIRA agent through the OpenAI Agents SDK over the Responses API.
   - Streams model events.
   - Continues through tool calls until a final answer, approval pause, cancellation, or bounded failure.
   - Tracks a durable run state.

3. **Context assembler**
   - Loads the versioned MIRA instruction contract.
   - Adds the active conversation and task state.
   - Retrieves only relevant memories.
   - Supplies active permissions and environment facts.
   - Avoids dumping the entire database into every prompt.

4. **Tool registry and tool search**
   - Keeps a catalog of all tools without loading every schema on every turn.
   - Exposes only the tools relevant to the current task.
   - Uses OpenAI tool search when appropriate.
   - Validates tool arguments with strict schemas.

5. **Policy and approval engine**
   - Classifies each tool call by side effect and risk.
   - Applies persistent user grants.
   - Pauses for approval when required.
   - Produces an immutable action receipt.

6. **Tool executor**
   - Executes local function tools.
   - Connects to approved MCP servers and service APIs.
   - Enforces timeouts, rate limits, scopes, idempotency, and cancellation.
   - Returns structured success or structured failure to the model.

7. **Memory service**
   - Stores short-term session state, episodic events, semantic facts/preferences, and procedural knowledge separately.
   - Supports recall, correction, provenance, expiration, and deletion.
   - Never treats a model guess as a confirmed fact.

8. **Run and job store**
   - Persists tasks, tool calls, approvals, retries, checkpoints, and final outcomes.
   - Allows safe restart after a crash.
   - Prevents duplicate side effects.

9. **Observability and evaluation**
   - Captures model calls, selected tools, tool arguments, tool outputs, policy decisions, latency, token use, and failures.
   - Redacts secrets.
   - Provides regression tests for agent behavior.

### 5.2 Recommended repository layout

```text
Mira/
  README.md
  pyproject.toml
  .env.example
  prompts/
    mira_core.md
    tool_policy.md
  src/mira/
    app.py
    config.py
    agent/
      runtime.py
      context.py
      state.py
      events.py
    policy/
      engine.py
      approvals.py
      grants.py
      receipts.py
    tools/
      registry.py
      schemas.py
      hosted.py
      memory_tools.py
      local_files.py
      local_shell.py
      github.py
      google.py
      discord.py
      minecraft.py
    memory/
      models.py
      store.py
      retrieval.py
      consolidation.py
    transport/
      http.py
      websocket.py
      stt_adapter.py
      tts_adapter.py
    jobs/
      scheduler.py
      worker.py
      models.py
    observability/
      logging.py
      tracing.py
      metrics.py
  migrations/
  tests/
    unit/
    integration/
    evals/
  scripts/
    Start-MIRA.ps1
    Stop-MIRA.ps1
    Test-MIRA.ps1
  docs/
    MIRA_AGENTIC_CORE_PLAN.md
    TOOL_CATALOG.md
    PERMISSION_MODEL.md
    STT_TTS_CONTRACT.md
```

## 6. Agent Loop Contract

Every user turn becomes a durable run with this state machine:

```text
RECEIVED
  -> CONTEXT_READY
  -> MODEL_RUNNING
  -> TOOL_PROPOSED
  -> POLICY_CHECK
      -> APPROVAL_REQUIRED
      -> TOOL_RUNNING
  -> TOOL_OBSERVED
  -> MODEL_RUNNING (repeat as needed)
  -> RESPONDING
  -> COMPLETED
```

Alternate terminal states:

- `CANCELLED`
- `FAILED_RETRYABLE`
- `FAILED_FINAL`
- `DENIED_BY_POLICY`

Required loop behavior:

- Never claim an action succeeded until the tool reports success.
- Never silently discard a failed tool call.
- Give the model structured error information so it can repair arguments or choose another route.
- Cap retries per tool and per run.
- Cancel active tools when the user interrupts, where the underlying tool supports cancellation.
- Store an idempotency key before every write action.
- Resume a recoverable run after restart without repeating completed writes.
- Separate internal status events from the final text sent to TTS.

## 7. Tool Contract

Every application-owned tool must declare:

```text
name
version
description
strict input schema
strict output schema
capability tags
read/write classification
risk tier
required credential scope
approval mode
timeout
retry policy
idempotency behavior
reversibility
secret-handling policy
```

Example risk tiers:

| Tier | Meaning | Default behavior |
|---|---|---|
| 0 | no side effect: calculate, inspect, search | automatic |
| 1 | local, reversible, tightly scoped write | automatic only with an existing scoped grant |
| 2 | external write or communication | approval unless a persistent scoped grant exists |
| 3 | destructive, financial, account/security, broad shell/admin | explicit approval every time |
| 4 | prohibited by product policy | deny |

Approval choices should support:

- allow once;
- allow for this run;
- always allow this exact action scope;
- deny once;
- always deny this action scope.

A grant must be defined by tool, operation, target scope, expiry, and maximum risk. “Always allow GitHub” is too broad. “Allow MIRA to create issues in `crazytaxzi/Mira`” is a usable grant.

### Programmatic tool calling policy

Use programmatic tool calling for bounded, read-heavy work such as filtering, joining, ranking, aggregating, validating, and deduplicating tool results. Keep write actions and approval-sensitive actions as direct tool calls so the authorization boundary remains visible and auditable.

## 8. Memory Design

MIRA needs four distinct forms of memory.

### 8.1 Conversation/session memory

The exact recent dialogue and tool outcomes required to continue the current task. This is bounded and compacted when necessary.

### 8.2 Episodic memory

Timestamped events such as:

- what happened;
- who was involved;
- where the information came from;
- what action MIRA took;
- whether the result was verified;
- confidence and sensitivity;
- links to the related run and action receipt.

### 8.3 Semantic memory

Durable facts, preferences, rules, relationships, and stable project knowledge. Semantic memory must support superseding old values rather than accumulating contradictions forever.

### 8.4 Procedural memory

Versioned skills, tool-use instructions, workflows, and operating rules. These belong primarily in source control, not improvised memory records.

### 8.5 Memory rules

- User statements may become candidate memories.
- Tool-verified facts may become candidate memories with provenance.
- Model inference alone is not a confirmed memory.
- Sensitive memories require an explicit policy and may require confirmation.
- Every memory can be corrected, superseded, expired, or deleted.
- Retrieval must be relevance- and scope-based.
- MIRA must have explicit `remember`, `recall`, `correct`, and `forget` tools.

For the first local release, use SQLite in WAL mode with structured tables and full-text search. Introduce embeddings/vector retrieval only after the structured memory behavior is correct. Do not begin by hiding every bug inside a vector database.

## 9. STT and TTS Integration Contract

The existing speech systems remain separate services.

### STT -> MIRA

```json
{
  "event": "utterance.final",
  "session_id": "primary",
  "utterance_id": "uuid",
  "speaker_id": "senti",
  "text": "Find the latest MIRA errors and explain the likely cause.",
  "confidence": 0.94,
  "captured_at": "ISO-8601 timestamp",
  "source": "voicemeeter-b1"
}
```

### MIRA -> TTS/UI

```json
{
  "event": "assistant.output.delta",
  "run_id": "uuid",
  "text": "I found the first failure...",
  "speak": true
}
```

Other required events:

- `assistant.status`
- `tool.started`
- `tool.completed`
- `tool.failed`
- `approval.requested`
- `approval.resolved`
- `assistant.output.final`
- `run.cancelled`

Barge-in behavior:

1. STT detects a new deliberate utterance while TTS is speaking.
2. Voice bridge stops TTS.
3. Active model stream is cancelled.
4. Cancellable tools receive cancellation.
5. Non-cancellable writes finish safely and report their receipt.
6. The new utterance begins a new run with the interrupted run attached as context.

## 10. Security Boundaries

- Secrets live in environment variables or the operating-system credential store, never prompts, logs, Git history, or memory text.
- Tool outputs are untrusted data and may contain prompt injection.
- Web pages, emails, documents, and MCP output cannot alter the core instruction or permission policy.
- Local shell starts disabled and later becomes repository/workspace scoped.
- Computer use runs in an isolated browser/container with domain and action allowlists.
- Purchases, financial actions, credential changes, account changes, deletion, and broad administration always require explicit approval.
- External communications must identify the sender correctly and must not impersonate the user without authorization.
- Every external write produces an action receipt containing target, operation, timestamp, result, and remote identifier.
- Logs redact API keys, OAuth tokens, cookies, passwords, and sensitive memory values.

## 11. Delivery Milestones

### Milestone 0 — Repository and runtime foundation

Deliver:

- Python project and locked dependencies;
- configuration loader and `.env.example`;
- versioned MIRA core prompt;
- structured logging;
- SQLite database and migrations;
- health endpoint;
- Windows start, stop, and test scripts;
- CI for lint, type checking, and tests.

Acceptance:

- a clean machine can start MIRA with one documented command;
- missing configuration produces a precise error;
- secrets are not printed;
- health check reports model configuration, database status, and enabled tool groups without exposing credentials.

### Milestone 1 — Agent kernel vertical slice

Deliver:

- `gpt-5.6-luna` through the Responses API/Agents SDK;
- streaming text input/output;
- durable run state machine;
- strict tool registry;
- deterministic `calculator`, `current_time`, and `system_health` tools;
- one sandboxed reversible write tool;
- policy check, approval pause, and action receipts;
- timeout, retry, cancellation, and idempotency support.

Acceptance scenario:

> “Tell me the current time, calculate 44 × 20.9177, and save the result to the MIRA scratch workspace.”

MIRA must call the correct read tools, request approval for the write unless a matching grant exists, write only inside the sandbox, verify the write, and report the exact completed actions.

### Milestone 2 — Voice bridge

Deliver:

- STT event adapter;
- TTS streaming adapter;
- barge-in and run cancellation;
- separation of speakable output from status/tool events;
- reconnect and duplicate-utterance handling.

Acceptance:

- a spoken request completes end to end;
- a second utterance interrupts speech and cancels the first run cleanly;
- duplicate STT delivery does not duplicate actions.

### Milestone 3 — Durable memory

Deliver:

- session, episodic, semantic, and procedural stores;
- memory proposal and consolidation rules;
- recall with provenance;
- correction, supersession, expiration, and deletion;
- memory inspection endpoint/UI payloads.

Acceptance:

- a user-approved fact is recalled in a later process restart;
- a correction supersedes the old value;
- `forget` removes it from future retrieval;
- unrelated memories are not injected into unrelated turns.

### Milestone 4 — OpenAI-hosted capability tools

Deliver:

- web search with citations;
- file search and managed knowledge collections;
- code interpreter;
- image input;
- optional image generation;
- tool search/dynamic loading;
- programmatic tool calling for bounded read-heavy workflows.

Acceptance:

- current-information questions use web search and preserve citations;
- an uploaded document can be searched without dumping the full document into the prompt;
- a data-analysis request produces a verified calculation or artifact;
- tool schemas not relevant to the turn are not loaded unnecessarily.

### Milestone 5 — Local tool gateway

Deliver:

- workspace-scoped file read/write;
- process and service health tools;
- restricted local shell;
- repository-scoped code inspection and patching;
- optional isolated computer-use harness;
- persistent permission grants.

Acceptance:

- MIRA can diagnose a local service from logs and health data;
- shell cannot escape the configured workspace or read secret locations;
- code changes are shown as patches and verified by tests;
- destructive actions pause for explicit approval.

### Milestone 6 — External connectors

Add and verify one connector at a time:

1. GitHub;
2. Google Calendar/Contacts/Gmail/Drive as required;
3. Discord;
4. Twitch/OBS/music controls;
5. Minecraft administration;
6. any later household or work integrations.

Each connector requires:

- least-privilege credentials;
- read/write operation separation;
- explicit risk classification;
- integration tests using a safe target;
- action receipts;
- revocation instructions.

### Milestone 7 — Autonomous jobs and event-driven work

Deliver:

- durable jobs and checkpoints;
- webhook/event triggers;
- scheduled reminders without a polling busy-loop;
- bounded multi-step task execution;
- resume after restart;
- spend, time, and tool-call budgets;
- progress/status events.

Acceptance:

- a scheduled or event-triggered job survives restart;
- a failed dependency retries within policy;
- a write is never duplicated after recovery;
- MIRA stops at approval boundaries instead of inventing authorization.

### Milestone 8 — Production hardening

Deliver:

- trace inspection and redaction;
- agent behavior evaluation suite;
- prompt-injection and tool-abuse tests;
- fault injection for timeouts, malformed outputs, and partial failures;
- cost and latency metrics;
- backup/restore for memory and grants;
- Windows production packaging and operator guide.

Release gates:

- no unapproved tier-2 or tier-3 writes in the regression suite;
- no duplicate side effects during retry/restart tests;
- at least 95% success on the agreed representative task suite;
- all failures produce an honest, actionable final status;
- user can inspect and revoke permissions and memories.

## 12. First Bounded Product Target

The first implementation target is **MIRA Agentic Core v0.1**. It includes Milestones 0 and 1 only.

It deliberately does **not** begin with Discord, Minecraft, Gmail, full desktop control, or a giant MCP catalog. Those integrations depend on a reliable kernel. Bolting twenty tools onto a weak loop creates a louder failure, not a smarter agent.

### v0.1 backlog

- [ ] bootstrap Python project and CI;
- [ ] implement configuration and secret redaction;
- [ ] create the versioned core instruction contract;
- [ ] connect `gpt-5.6-luna` with streaming;
- [ ] implement durable run/event tables;
- [ ] implement strict tool registry;
- [ ] add calculator, current time, and system health tools;
- [ ] add sandbox file write tool;
- [ ] implement risk policy, approvals, grants, and receipts;
- [ ] implement retries, timeouts, cancellation, and idempotency;
- [ ] add unit and end-to-end tests for the acceptance scenario;
- [ ] add one-command Windows start/stop/test scripts.

## 13. Explicit Non-Goals for the First Release

- unrestricted administrator access;
- loading every available tool into every prompt;
- autonomous purchases or financial transactions;
- silent impersonation of the user;
- self-modifying production code without review;
- replacing the existing STT or TTS engines;
- fine-tuning before prompt, tool, memory, and evaluation failures are understood;
- splitting MIRA into multiple personalities or independent agents.

MIRA remains one primary agent. Later, bounded subagents may be used internally for parallel research or specialist analysis, but they do not become separate identities and they do not bypass MIRA's policy engine.

## 14. Official Technical References

- [GPT-5.6 Luna model](https://developers.openai.com/api/docs/models/gpt-5.6-luna)
- [Using tools](https://developers.openai.com/api/docs/guides/tools)
- [Responses API migration](https://developers.openai.com/api/docs/guides/migrate-to-responses)
- [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/)
- [Conversation state](https://developers.openai.com/api/docs/guides/conversation-state)
- [Programmatic tool calling](https://developers.openai.com/api/docs/guides/tools-programmatic-tool-calling)
- [MCP and connectors](https://developers.openai.com/api/docs/guides/tools-connectors-mcp)
- [Secure MCP Tunnel](https://developers.openai.com/api/docs/guides/secure-mcp-tunnels)
- [Guardrails and human review](https://developers.openai.com/api/docs/guides/agents/guardrails-approvals)
- [Evaluate agent workflows](https://developers.openai.com/api/docs/guides/agent-evals)
