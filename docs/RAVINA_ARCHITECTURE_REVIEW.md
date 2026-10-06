# Ravina Architecture Review of DS2API

## Purpose

This document evaluates `ds2api` as an architecture reference for the Ravina AI Gateway.

The goal is **not** to copy DS2API into Ravina. The upstream project is AGPL-3.0 licensed, relies on reverse-engineered DeepSeek Web behavior, and has a different trust/security model from Ravina. Instead, this review separates architectural ideas into:

- **ADOPT** — concepts that fit Ravina and should be implemented independently.
- **ADAPT** — concepts that are useful but require redesign for Ravina's multi-tenant SaaS architecture.
- **REJECT** — implementation choices that should not enter Ravina.

This review is based on the current fork:

`tandis/ds2api`

Primary code areas reviewed:

- `internal/promptcompat`
- `internal/completionruntime`
- `internal/account`
- `internal/auth`
- `internal/httpapi`
- `internal/toolcall`
- `internal/toolstream`
- `internal/stream`
- `internal/deepseek`
- `config.example.json`
- `go.mod`
- `SECURITY.md`

---

## Executive Decision

DS2API is valuable as a **protocol/runtime architecture reference**, but it should not become a runtime dependency of Ravina Smart Direct.

Recommended direction:

```text
DS2API
  ↓
Study architecture
  ↓
Extract contracts and patterns
  ↓
Write Ravina-native specification
  ↓
Clean-room implementation
  ↓
Ravina AI Gateway
```

The strongest transferable ideas are:

1. Canonical request normalization
2. Canonical response normalization
3. Provider-independent execution runtime
4. Capacity-aware credential pooling
5. Retry orchestration
6. Canonical streaming events
7. Tool-call normalization and anti-leak handling
8. Protocol adapters at the edge

The weakest or unsafe-to-copy areas are:

1. Reverse-engineered DeepSeek Web dependency
2. Passing unknown API keys through as upstream credentials
3. Plain config-based account password/token storage
4. Parser-first tool execution
5. Product architecture centered on pooled upstream website accounts
6. Any direct code reuse that creates AGPL obligations for Ravina proprietary code

---

# 1. Architecture Comparison

## DS2API

```text
OpenAI / Claude / Gemini Client
             ↓
        HTTP Adapters
             ↓
       promptcompat
             ↓
     StandardRequest
             ↓
        Auth Resolver
             ↓
        Account Pool
             ↓
    Completion Runtime
             ↓
   DeepSeek Web Protocol
             ↓
      SSE / Streaming
             ↓
 AssistantTurn Normalizer
             ↓
 OpenAI / Claude / Gemini
```

## Recommended Ravina target

```text
Instagram / Internal Agent / API Client
                  ↓
           Protocol Adapters
                  ↓
          Canonical AI Request
                  ↓
               Router
                  ↓
      Tenant Credential Resolution
                  ↓
          Provider Adapter
                  ↓
         Execution Runtime
                  ↓
       Canonical Stream Events
                  ↓
     Canonical Assistant Response
                  ↓
       Tool / Action Dispatcher
                  ↓
   Conversation / Sales Orchestrator
```

Critical difference:

DS2API is primarily an API compatibility layer around a single reverse-engineered upstream service.

Ravina is a multi-tenant **AI application platform** where provider access is only one layer beneath business logic such as conversation state, lead qualification, next-best-action, and handoff.

---

# 2. ADOPT

## ADOPT-01 — Canonical Request Model

### DS2API reference

`internal/promptcompat` normalizes different request formats before execution.

Relevant files include:

- `standard_request.go`
- `request_normalize.go`
- `message_normalize.go`
- `responses_input_normalize.go`
- `prompt_build.go`
- `tool_prompt.go`

### Ravina decision

Adopt the pattern, not the code.

Create a provider-independent request contract.

Suggested shape:

```ts
interface CanonicalAIRequest {
  tenantId: string;
  capability: "chat" | "embedding" | "vision" | "tool_call";
  modelHint?: string;
  messages: CanonicalMessage[];
  tools?: CanonicalToolDefinition[];
  toolChoice?: CanonicalToolChoice;
  stream: boolean;
  temperature?: number;
  metadata?: Record<string, unknown>;
}
```

Benefits:

- Provider adapters become replaceable.
- Mistral/Groq/OpenRouter behavior is isolated.
- Business logic does not depend on SDK-specific request shapes.
- Protocol compatibility can be added later without contaminating provider code.

Priority: **High**

---

## ADOPT-02 — Canonical Response Model

DS2API uses `assistantturn` to normalize output semantics before serialization.

Ravina should do the same.

Suggested result contract:

```ts
interface CanonicalAIResponse {
  provider: string;
  model: string;
  text: string;
  finishReason: string;
  usage?: TokenUsage;
  toolCalls?: CanonicalToolCall[];
  reasoningMetadata?: Record<string, unknown>;
  providerRequestId?: string;
}
```

The same principle should apply to failures:

```ts
interface CanonicalProviderError {
  category:
    | "auth"
    | "rate_limit"
    | "quota"
    | "timeout"
    | "provider_unavailable"
    | "model_unavailable"
    | "bad_request"
    | "content_policy"
    | "unknown";

  retryable: boolean;
  provider: string;
  model?: string;
  statusCode?: number;
  retryAfterMs?: number;
}
```

Priority: **High**

---

## ADOPT-03 — Separate Protocol Edge from Execution Core

DS2API keeps OpenAI/Claude/Gemini HTTP surfaces separate from the runtime.

Ravina should preserve the same separation:

```text
HTTP / Internal API
      ↓
Protocol adapter
      ↓
Canonical contract
      ↓
Execution core
```

Do not let OpenAI-specific fields leak into provider routing or sales-agent logic.

Priority: **High**

---

## ADOPT-04 — Execution Runtime as an Orchestration Layer

DS2API's `completionruntime` is a useful boundary.

The runtime coordinates:

- request preparation
- session/preflight work
- provider execution
- retry
- failover
- response collection
- normalization

Ravina should have a similar provider-neutral runtime:

```text
ExecutionRuntime.execute(request)
   ↓
Resolve route
   ↓
Lease credential
   ↓
Invoke provider
   ↓
Normalize result
   ↓
Apply retry/failover policy
   ↓
Emit telemetry
```

Priority: **High**

---

## ADOPT-05 — Capacity-Aware Credential Leasing

DS2API's `internal/account` separates account acquisition/release from request execution.

Useful concepts:

- per-account inflight limit
- global inflight limit
- wait queue
- release semantics
- exclusion set for retries
- queue rotation

Ravina should implement this against **tenant-scoped provider credentials**, not shared website accounts.

Suggested abstraction:

```ts
interface CredentialLease {
  credentialId: string;
  provider: string;
  tenantId: string;
  release(): Promise<void>;
}
```

Selection inputs should include:

- tenant ownership
- provider
- capability
- credential health
- quota pool state
- cooldown
- current inflight
- latency/reliability score

Priority: **Medium-High**

---

## ADOPT-06 — Retry State Must Be Explicit

DS2API tracks attempted accounts and prevents retry loops.

Ravina should formalize this as request execution state:

```ts
interface ExecutionAttemptState {
  attemptedCredentialIds: Set<string>;
  attemptedProviders: Set<string>;
  attemptCount: number;
  startedAt: number;
  maxAttempts: number;
}
```

This avoids accidental cycling and makes telemetry explainable.

Priority: **High**

---

## ADOPT-07 — Streaming Normalization

DS2API isolates SSE consumption from protocol serialization.

Ravina should expose provider-independent streaming events:

```ts
type CanonicalStreamEvent =
  | { type: "text_delta"; text: string }
  | { type: "tool_call_start"; id: string; name: string }
  | { type: "tool_call_delta"; id: string; argumentsDelta: string }
  | { type: "tool_call_end"; id: string }
  | { type: "usage"; usage: TokenUsage }
  | { type: "completed"; finishReason: string }
  | { type: "error"; error: CanonicalProviderError };
```

This allows:

- Mistral streaming
- OpenRouter streaming
- Groq streaming
- internal UI events
- future OpenAI-compatible API surface

to share one semantic stream.

Priority: **Medium-High**

---

## ADOPT-08 — Request Size Limits

DS2API applies body-size limits before JSON decoding.

Ravina should enforce:

- HTTP body limit
- attachment limits
- maximum tool schema size
- maximum context size
- per-tenant usage limits

Priority: **High**

---

## ADOPT-09 — Regression Fixtures for Tool Calling

DS2API has extensive tool-call regression tests.

Ravina should build a fixture suite covering:

- valid native tool call
- malformed arguments
- partial streaming arguments
- unknown tool
- missing required field
- tool call emitted as plain text
- nested JSON
- duplicate tool calls
- cancellation mid-stream
- malicious prompt attempting fake tool syntax

Priority: **High before agent actions become production-critical**

---

# 3. ADAPT

## ADAPT-01 — Account Pool → Tenant Credential Pool

DS2API pools DeepSeek website accounts globally.

Ravina must never generalize that design directly.

Target:

```text
Tenant A
  ↓
Mistral Credential A1
Mistral Credential A2

Tenant B
  ↓
Mistral Credential B1

NO cross-tenant fallback
```

Hard invariant:

```text
credential.tenant_id MUST equal request.tenant_id
```

Provider failover may cross providers **inside the same tenant only**.

Priority: **Critical**

---

## ADAPT-02 — Retry Policy

DS2API performs account switching on certain 429 conditions.

Ravina needs error-category-aware retry:

```text
model-specific error
   ↓
same provider / alternate model

credential-specific auth failure
   ↓
same tenant / alternate credential

provider-wide outage
   ↓
alternate provider

tenant quota exhausted
   ↓
do not borrow another tenant's capacity
```

Retry should consider:

- idempotency
- streaming state
- whether output has already been emitted
- tool execution side effects
- provider Retry-After
- remaining latency budget

Priority: **Critical**

---

## ADAPT-03 — Round-Robin → Scored Candidate Routing

DS2API uses simple queue rotation.

Ravina already needs richer routing.

Candidate score should incorporate:

```text
capability match
tenant credential availability
quota health
reliability
latency
cost
cooldown
provider health
model preference
```

Round-robin can remain a tie-breaker only.

Priority: **Medium**

---

## ADAPT-04 — Tool Call Normalization

DS2API must parse tool syntax from model text because of its web-chat compatibility layer.

Ravina should instead use:

```text
Native provider tool call
         ↓
Canonical ToolCall
         ↓
Schema validation
         ↓
Policy authorization
         ↓
Tool executor
```

Text parsing is only a compatibility fallback.

Priority: **Critical for Sales Agent actions**

---

## ADAPT-05 — Anti-Leak Tool Streaming

DS2API buffers suspicious tool syntax so raw markup is not leaked.

Ravina should preserve the security objective but implement it semantically:

- tool call content must never be sent to the customer as ordinary text
- action arguments must be validated before execution
- tool execution output must be filtered before becoming customer-facing content
- internal action metadata must remain internal

This is especially important for:

- `create_lead`
- `book_demo`
- `handoff_to_sales`
- `send_catalog`
- `schedule_followup`

Priority: **Critical**

---

## ADAPT-06 — Authentication Boundary

DS2API accepts unknown bearer values as direct upstream DeepSeek tokens.

Ravina must instead distinguish:

```text
Ravina API credential
Provider credential
Internal service identity
```

No ambiguous fallback.

Recommended behavior:

```text
unknown Ravina API key
   ↓
401
```

Provider credentials must be resolved only from the encrypted tenant credential store.

Priority: **Critical**

---

## ADAPT-07 — Configuration Hot Reload

DS2API supports runtime configuration changes.

Ravina should retain hot-reload semantics where useful, but configuration must be split into:

- platform configuration
- provider configuration
- tenant configuration
- encrypted credentials
- routing policy
- feature flags

Do not use one JSON file as the source of truth in production.

Priority: **Medium**

---

# 4. REJECT

## REJECT-01 — Reverse-Engineered DeepSeek Web as Core Production Provider

Do not make Ravina depend on:

- DeepSeek Web session creation
- PoW
- website authentication
- reverse-engineered transport
- undocumented web endpoints

Official provider APIs should be preferred.

Reason:

- high breakage risk
- account suspension risk
- policy/terms risk
- no production SLA
- upstream behavior can change without notice

Decision: **Reject for Ravina production core**

---

## REJECT-02 — Unknown API Key as Upstream Token

DS2API behavior:

```text
key not found in DS2API config
   ↓
treat key as DeepSeek token
```

This is incompatible with Ravina's trust boundary.

Decision: **Reject**

---

## REJECT-03 — Plain Config Password Storage

`config.example.json` models account credentials using email/password fields.

Ravina must use encrypted tenant credential storage.

Current Ravina direction:

```text
Tenant Credential
   ↓
AES-GCM
   ↓
encrypted storage
   ↓
decrypt only during provider execution
```

Decision: **Reject**

---

## REJECT-04 — Cross-Tenant Credential Borrowing

Under no circumstance should one tenant's key/account satisfy another tenant's request.

Decision: **Reject permanently**

---

## REJECT-05 — Parser-First Tool Execution

Tool execution must not depend primarily on recognizing XML/DSML-like text in model output.

Decision: **Reject as primary architecture**

Fallback parser may exist only in isolated compatibility code.

---

## REJECT-06 — Direct Code Copy into Proprietary Ravina Core

DS2API is AGPL-3.0.

This review intentionally extracts architectural concepts rather than copying implementation.

Any code reuse must be separately evaluated for license obligations.

Decision: **Reject direct copy/paste**

---

# 5. Ravina AI Gateway Target Architecture

```text
                         RAVINA AI GATEWAY
                                │
              ┌─────────────────┴─────────────────┐
              │                                   │
       Internal Agent API                 Compatibility APIs
                                                (future)
              │                                   │
              └─────────────────┬─────────────────┘
                                ↓
                     Canonical Request Layer
                                │
                                ↓
                           AI Router
                                │
            ┌───────────────────┼───────────────────┐
            ↓                   ↓                   ↓
        Mistral             Groq              OpenRouter
        Adapter             Adapter              Adapter
            │                   │                   │
            └───────────────────┼───────────────────┘
                                ↓
                       Execution Runtime
                                │
                 ┌──────────────┼──────────────┐
                 ↓              ↓              ↓
              Retry          Quota          Telemetry
              Policy         Policy
                 │              │
                 └───────┬──────┘
                         ↓
                Canonical Response
                         │
            ┌────────────┴────────────┐
            ↓                         ↓
      Canonical Stream          Canonical ToolCall
            │                         │
            ↓                         ↓
       Customer Reply            Action Policy
                                      │
                                      ↓
                                Tool Executor
                                      │
               ┌──────────────────────┼──────────────────────┐
               ↓                      ↓                      ↓
          Lead Engine          Human Handoff          Booking/CRM
```

---

# 6. Proposed Ravina Modules

Recommended modular-monolith layout:

```text
lib/
└── ai-gateway/
    ├── contracts/
    │   ├── canonical-request.js
    │   ├── canonical-response.js
    │   ├── canonical-error.js
    │   ├── canonical-stream-event.js
    │   └── canonical-tool-call.js
    │
    ├── protocols/
    │   ├── internal/
    │   ├── openai/          # future
    │   ├── anthropic/       # future
    │   └── gemini/          # future
    │
    ├── runtime/
    │   ├── execution-runtime.js
    │   ├── attempt-state.js
    │   ├── retry-policy.js
    │   └── stream-runtime.js
    │
    ├── routing/
    │   ├── router.js
    │   ├── candidate-score.js
    │   └── provider-health.js
    │
    ├── credentials/
    │   ├── credential-store.js
    │   ├── credential-pool.js
    │   ├── lease.js
    │   └── quota-pool.js
    │
    ├── providers/
    │   ├── mistral/
    │   ├── groq/
    │   └── openrouter/
    │
    ├── tools/
    │   ├── normalize.js
    │   ├── validate.js
    │   └── policy.js
    │
    └── telemetry/
        ├── events.js
        └── metrics.js
```

Keep this a **modular monolith** for now.

Do not split it into microservices before load and organizational boundaries justify that complexity.

---

# 7. Integration with Ravina Sales Agent Roadmap

Gateway work must support, not displace, the product roadmap.

Current product priority remains:

```text
Conversation State Engine
        ↓
Lead Profile Foundation
        ↓
Lead Qualification
        ↓
Lead Scoring
        ↓
Next Best Action
        ↓
Human Handoff
        ↓
Operator Workspace
```

Therefore DS2API-inspired gateway improvements should be introduced incrementally.

Recommended rule:

> No gateway refactor should delay the next end-to-end Sales Agent milestone unless it removes a direct blocker.

---

# 8. Recommended Delivery Order

## Phase A — Contracts

Implement:

- CanonicalAIRequest
- CanonicalAIResponse
- CanonicalProviderError
- CanonicalToolCall
- CanonicalStreamEvent

No behavior changes yet.

## Phase B — Runtime Boundary

Move current provider execution behind:

`ExecutionRuntime`

Keep Mistral behavior unchanged.

## Phase C — Credential Leasing

Introduce tenant-scoped credential lease abstraction.

Preserve current isolation and encryption.

## Phase D — Error Taxonomy + Retry Policy

Centralize:

- rate limit
- quota
- auth
- timeout
- provider outage
- model unavailable

## Phase E — Tool Contract

Prepare structured Agent Actions for the Sales Agent.

## Phase F — Optional Provider Expansion

Only after core Sales Agent milestones:

- Groq
- OpenRouter

---

# 9. Acceptance Criteria for Clean-Room Implementation

The Ravina implementation is acceptable only if all of the following hold:

1. No DS2API source file is copied into Ravina.
2. No cross-tenant credential fallback is possible.
3. Unknown Ravina API credentials return 401.
4. Raw provider credentials never appear in logs.
5. Provider-specific request/response types do not escape provider adapters.
6. Retry loops are bounded.
7. Every retry records reason and candidate.
8. Tool calls are schema validated before execution.
9. Tool calls require policy authorization.
10. Streaming tool syntax cannot leak into customer text.
11. Human handoff actions are idempotent.
12. Provider failure does not corrupt conversation state.
13. Existing Mistral behavior remains regression-tested.
14. Gateway refactor does not alter the deterministic lead-scoring design.
15. AI auto-send remains forbidden while conversation ownership is human-active.

---

# 10. Final Classification

| DS2API concept | Ravina decision |
|---|---|
| Canonical request normalization | ADOPT |
| Canonical response normalization | ADOPT |
| Protocol adapters | ADOPT |
| Execution runtime boundary | ADOPT |
| Streaming normalization | ADOPT |
| Explicit retry attempt state | ADOPT |
| Regression fixtures | ADOPT |
| Account pool | ADAPT → tenant credential pool |
| Round-robin | ADAPT → scored routing |
| 429 alternate-account retry | ADAPT → typed failover policy |
| Tool-call parser | ADAPT → native structured tools first |
| Anti-leak buffering | ADAPT → semantic tool isolation |
| Hot config reload | ADAPT |
| DeepSeek Web reverse engineering | REJECT |
| PoW/web-session dependency | REJECT |
| Unknown key as upstream token | REJECT |
| Plain password config | REJECT |
| Cross-tenant borrowing | REJECT |
| Parser-first execution | REJECT |
| Direct AGPL code copy | REJECT |

---

# 11. Next Engineering Artifact

The next artifact should be a clean-room implementation specification:

`docs/RAVINA_AI_GATEWAY_CLEANROOM_SPEC.md`

It should define:

- contracts
- interfaces
- invariants
- state transitions
- error taxonomy
- retry/failover policy
- credential lease lifecycle
- tool-call lifecycle
- telemetry events
- test matrix
- migration strategy from the current Ravina Gateway

That document can then be given directly to Codex as the implementation contract.

---

## Final Recommendation

Use DS2API as a **reference implementation for gateway engineering discipline**, not as Ravina infrastructure.

The design goal is:

```text
Learn from DS2API
      ↓
Do not inherit its trust model
      ↓
Do not inherit its reverse-engineered upstream
      ↓
Do not copy AGPL implementation
      ↓
Build Ravina-native canonical gateway contracts
      ↓
Keep product focus on AI Sales Agent
```
