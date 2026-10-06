
# Ravina AI Gateway Clean-room Implementation Specification

## Status

Implementation contract for Codex and engineering work.

## Scope

This specification defines a clean-room implementation plan for evolving the existing Ravina AI Gateway without copying DS2API source code.

The design is informed by architecture patterns observed in tandis/ds2api, but this document defines Ravina-native contracts, invariants, runtime behavior, tests, and migration steps.

Target environment:
- Node.js 22
- Express 5
- PostgreSQL
- multi-tenant architecture
- encrypted tenant provider credentials
- Mistral as the currently active provider
- future optional providers: Groq and OpenRouter
- current business capabilities: Direct Reply, Form Builder, Embeddings, Intent Detection, Lead Qualification

The implementation MUST remain a modular monolith unless a separate service boundary becomes operationally necessary.

---

# 1. Clean-room Rules

1. Do not copy DS2API source code.
2. Do not port DS2API functions line-by-line.
3. Do not reproduce DS2API parser implementation, comments, or tests verbatim.
4. Use this document as the implementation contract.
5. Provider-specific behavior must remain behind Ravina provider adapters.
6. Tenant isolation must be preserved at every layer.
7. No credential may cross tenant boundaries.
8. Unknown Ravina API credentials must fail closed.
9. Secrets must never appear in logs, telemetry, traces, or returned errors.
10. Existing Mistral behavior must remain functional throughout migration.

---

# 2. Target Architecture

Channel / Internal Feature / API Client
→ Feature or Protocol Adapter
→ Canonical AI Request
→ AI Router
→ Tenant Credential Resolver
→ Credential Lease
→ Provider Adapter
→ Execution Runtime
→ Retry / Failover / Streaming / Telemetry
→ Canonical AI Response
→ Tool and Action Dispatcher
→ Conversation / Lead / Handoff Logic

The AI Gateway is infrastructure for Ravina product capabilities. It must not become the product boundary itself.

---

# 3. Recommended Module Layout

src/lib/ai-gateway/

- contracts/
  - canonical-request.js
  - canonical-response.js
  - canonical-error.js
  - canonical-stream-event.js
  - canonical-tool-call.js
  - canonical-message.js

- runtime/
  - execution-runtime.js
  - execution-context.js
  - attempt-state.js
  - retry-policy.js
  - failover-policy.js
  - stream-runtime.js

- routing/
  - router.js
  - route-candidate.js
  - candidate-score.js
  - capability-map.js
  - provider-health.js

- credentials/
  - credential-store.js
  - credential-resolver.js
  - credential-lease.js
  - credential-pool.js
  - quota-state.js

- providers/
  - provider-adapter.js
  - mistral/
  - groq/
  - openrouter/

- tools/
  - tool-registry.js
  - tool-normalizer.js
  - tool-validator.js
  - tool-policy.js
  - tool-executor.js
  - idempotency.js

- telemetry/
  - events.js
  - metrics.js
  - tracing.js
  - redaction.js

- protocols/
  - internal/
  - openai/ future
  - anthropic/ future
  - gemini/ future

- testing/
  - fixtures/
  - fake-provider.js
  - assertions.js

If the current Ravina repository uses a different folder convention, preserve that convention while keeping the same logical boundaries.

---

# 4. Canonical Contracts

## 4.1 CanonicalMessage

Required fields:
- role: system | user | assistant | tool
- content: string

Optional:
- name
- toolCallId
- metadata

Rules:
- provider-native message objects must be converted before entering runtime
- runtime must not depend on provider SDK message classes
- message order must be preserved
- tool-result messages must reference a tool call ID

## 4.2 CanonicalAIRequest

Required:
- requestId
- tenantId
- capability
- messages for chat/tool calls

Optional:
- modelHint
- providerHint
- stream
- temperature
- maxTokens
- tools
- toolChoice
- metadata.conversationId
- metadata.leadId
- metadata.channel
- metadata.feature
- metadata.traceId

Capabilities:
- chat
- embedding
- vision
- tool_call

Hard rule:
Tenant identity comes from Ravina application context, never from provider credentials.

## 4.3 CanonicalToolDefinition

Fields:
- name
- description
- inputSchema
- riskLevel: low | medium | high
- idempotency: required | recommended | none

Recommended Ravina tool names:
- create_lead
- update_lead_profile
- qualify_lead
- schedule_followup
- book_demo
- send_catalog
- handoff_to_sales
- lookup_product
- lookup_price
- lookup_faq

Do not expose internal database function names as public tool names.

## 4.4 CanonicalToolCall

Fields:
- id
- name
- arguments
- providerRawId optional

Execution pipeline:
provider output
→ normalize
→ CanonicalToolCall
→ schema validation
→ policy authorization
→ idempotency check
→ execute

Raw provider output must never execute directly.

## 4.5 CanonicalAIResponse

Fields:
- requestId
- provider
- model
- text
- finishReason
- usage optional
- toolCalls optional
- providerRequestId optional
- metadata optional

finishReason values:
- stop
- length
- tool_call
- content_filter
- cancelled
- error
- unknown

## 4.6 CanonicalProviderError

Categories:
- auth
- rate_limit
- quota
- timeout
- provider_unavailable
- model_unavailable
- bad_request
- content_policy
- network
- cancelled
- unknown

Fields:
- category
- provider
- model optional
- statusCode optional
- providerCode optional
- retryable
- retryAfterMs optional
- safeMessage
- internalCause internal-only

internalCause must never be serialized to external clients.

---

# 5. Execution Context

Every execution creates an ExecutionContext containing:
- request
- startedAt
- deadlineAt optional
- attempts
- emittedOutput
- toolSideEffectsStarted
- cancellationSignal optional

Cancellation behavior:
- abort provider request
- stop stream
- release credential lease
- do not start new retries
- do not start new tools
- emit terminal telemetry

---

# 6. Attempt Tracking

Each attempt records:
- attemptNo
- provider
- model
- credentialId
- startedAt
- endedAt
- result
- errorCategory
- statusCode
- outputEmitted

Global attempt state tracks:
- attemptedCredentialIds
- attemptedProviderModels
- attemptCount

Invariant:
The same exact route candidate must not be retried indefinitely.

---

# 7. Provider Adapter Interface

Every provider adapter must implement the same semantic interface:

- name
- supports(capability)
- execute(request, context)
- stream(request, context) when supported
- mapError(error)
- healthCheck() optional

Provider adapters may use provider SDKs.

No module outside providers/* may depend on provider SDK response or error types.

---

# 8. Credential Store

Credential record must include:
- id
- tenantId
- provider
- encryptedSecret
- keyVersion
- enabled
- createdAt
- updatedAt
- metadata optional

Requirements:
- AES-GCM encryption remains enforced
- every lookup includes tenantId
- decrypted secret lifetime is minimal
- no global decrypted-secret cache
- no secrets in logs
- no secrets in telemetry
- no secrets in exception messages

---

# 9. Credential Lease Lifecycle

resolve candidates
→ acquire lease
→ increment inflight
→ decrypt credential
→ provider execution
→ update health/quota
→ release lease
→ decrement inflight

CredentialLease contains:
- credentialId
- tenantId
- provider
- secret
- release(outcome)

release() must be idempotent and safe to call more than once.

---

# 10. Credential Invariants

INV-CRED-01:
credential.tenantId must equal request.tenantId.

INV-CRED-02:
Cross-tenant fallback is forbidden.

INV-CRED-03:
Disabled credentials are never eligible.

INV-CRED-04:
Credentials in cooldown are not eligible.

INV-CRED-05:
Selection respects inflight capacity.

INV-CRED-06:
Health state must never expose secrets.

---

# 11. Routing

A RouteCandidate includes:
- provider
- model
- credentialId
- capabilityMatched
- healthScore
- latencyScore
- quotaScore
- costScore
- preferenceScore
- finalScore

V1 routing should stay simple.

Hard filters:
- tenant match
- credential enabled
- capability match
- not exhausted
- not in cooldown

Then prefer:
- explicit model/provider preference
- available capacity
- healthy provider
- low recent error rate

Tie-breaker:
- least inflight

Do not over-engineer cost and latency weighting until real telemetry exists.

---

# 12. Retry Policy

Never auto-retry when:
- request is invalid
- Ravina authentication failed
- content policy rejected the request
- tenant quota is exhausted
- cancellation occurred
- a non-idempotent tool side effect started
- customer-visible output was already emitted and replay could duplicate text
- maximum attempts reached

Auth failure:
provider auth failure
→ mark credential unhealthy
→ try alternate credential from the same tenant
→ fail if none exists

Rate limit:
429
→ respect Retry-After only when within latency budget
→ otherwise try alternate same-tenant credential
→ otherwise try alternate configured provider route
→ otherwise return canonical rate-limit error

Provider-wide failure:
5xx/network provider-wide condition
→ do not blindly rotate credentials
→ prefer provider failover when configured

Model unavailable:
→ configured model fallback
→ then provider fallback if allowed

All retry decisions must be deterministic and unit-tested.

---

# 13. Failover

Failover order must be configuration-driven.

Initial production configuration should contain only the current Mistral route.

Future example:
1. Mistral / mistral-small-latest
2. Groq / configured model
3. OpenRouter / configured model

Do not add providers only to increase architecture complexity.

---

# 14. Streaming

Canonical event types:
- start
- text_delta
- tool_call_start
- tool_call_delta
- tool_call_end
- usage
- completed
- error

Rules:
- provider-native SSE never leaves adapter boundary
- exactly one terminal event
- tool-call deltas never become customer-visible text
- cancellation closes the iterator
- error terminates the stream
- provider duplicate completion must not create duplicate terminal events

---

# 15. Tool Runtime

Lifecycle:
model emits structured tool call
→ normalize
→ lookup registered tool
→ schema validation
→ authorization
→ idempotency check
→ execute
→ persist outcome
→ return safe tool result to model

Unknown tool names are rejected.

Arbitrary function names from model output must never be executed dynamically.

---

# 16. Tool Registry

Each registered tool has:
- definition
- authorize(context)
- execute(args, context)

Authorization inputs may include:
- tenantId
- conversationId
- leadId
- current conversation owner
- channel
- feature enablement
- state machine state
- operator override
- risk level

---

# 17. Human Handoff Invariants

INV-HANDOFF-01:
When conversation ownership is human-active, AI auto-send is forbidden.

INV-HANDOFF-02:
handoff_to_sales must be idempotent.

INV-HANDOFF-03:
Repeated handoff calls must not create duplicate tickets or assignments.

Recommended ownership states:
- ai
- pending_handoff
- human
- paused

---

# 18. Lead Scoring Boundary

LLM may extract signals such as:
- intent
- budget mention
- timeline
- product interest
- contact information
- urgency
- objection category

Final lead score remains deterministic application logic.

Required flow:
LLM extracts facts
→ validate structured signals
→ deterministic scoring engine

Do not allow provider prose to directly set final lead score.

---

# 19. Error Mapping

Minimum mapping:

- provider 401/403 → auth
- 429 → rate_limit
- explicit quota exhaustion → quota
- socket timeout → timeout
- DNS/connectivity → network
- 500/502/503 → provider_unavailable
- invalid model → model_unavailable
- invalid request → bad_request
- policy rejection → content_policy
- AbortSignal → cancelled
- unknown failure → unknown

Unknown errors fail conservatively.

---

# 20. Provider Health and Circuit Breaker

Initial health state:
- healthy
- consecutiveFailures
- cooldownUntil
- lastFailureCategory
- lastSuccessAt
- lastFailureAt

Circuit breaker:
N consecutive provider-wide failures
→ open circuit
→ cooldown
→ allow probe
→ close on success

Credential-specific auth errors must not open the provider-wide circuit.

---

# 21. Telemetry

Required events:
- ai.request.received
- ai.route.selected
- ai.credential.leased
- ai.provider.attempt.started
- ai.provider.attempt.succeeded
- ai.provider.attempt.failed
- ai.retry.scheduled
- ai.failover.selected
- ai.stream.started
- ai.stream.completed
- ai.tool.requested
- ai.tool.authorized
- ai.tool.rejected
- ai.tool.executed
- ai.request.completed
- ai.request.failed

Common fields:
- timestamp
- requestId
- traceId
- tenantId
- provider
- model
- credentialId
- attemptNo
- durationMs
- metadata

Never include:
- API keys
- tokens
- passwords
- decrypted secrets
- raw Authorization headers
- full customer message by default

---

# 22. Redaction

Central redaction must cover key names containing:
- authorization
- api_key
- apikey
- token
- access_token
- refresh_token
- password
- secret
- credential

Replacement value:
[REDACTED]

Redaction runs before structured logging.

---

# 23. Metrics

Initial metrics:
- ai_requests_total
- ai_request_duration_ms
- ai_provider_attempts_total
- ai_provider_errors_total
- ai_retry_total
- ai_failover_total
- ai_rate_limit_total
- ai_tool_calls_total
- ai_tool_failures_total
- ai_stream_cancel_total

Useful dimensions:
- provider
- model
- capability
- result
- error category

Avoid high-cardinality tenant labels in Prometheus-style metrics unless infrastructure explicitly supports them.

---

# 24. Idempotency

Any side-effecting tool requires an idempotency key.

Recommended logical key:
tenantId + conversationId + toolName + logicalActionId

Persist the execution result.

Duplicate key:
return prior result instead of repeating side effect.

Critical tools:
- booking
- handoff
- lead creation
- outbound follow-up creation

---

# 25. Authentication Boundary

Caller authentication and provider credentials are separate trust domains.

Caller Credential
→ Ravina Auth
→ Tenant Context
→ AI Gateway
→ Encrypted Provider Credential

Unknown Ravina key:
401 Unauthorized

Never interpret an unknown caller bearer token as a provider token.

---

# 26. Tenant Isolation Tests

TEST-TENANT-01:
Tenant A cannot resolve Tenant B credential.

TEST-TENANT-02:
Tenant A cannot fail over to Tenant B credential.

TEST-TENANT-03:
Any credential cache key includes tenant identity.

TEST-TENANT-04:
Concurrent tenants cannot cross-contaminate request context, credentials, conversations, tool results, or lead state.

TEST-TENANT-05:
Logs contain credential IDs only, never credential values.

---

# 27. Retry Test Matrix

Required:
- 401 on credential A, credential B valid → use B
- 401 on only credential → fail auth
- 429 with alternate credential → use alternate credential
- 429 with Retry-After inside budget → bounded wait
- 429 with no capacity → canonical rate-limit error
- provider 503 → provider failover only if configured
- invalid request → no retry
- model unavailable → model/provider fallback
- cancelled request → no retry
- output already streamed → no unsafe transparent replay
- tool side effect started → no unsafe retry

---

# 28. Streaming Test Matrix

Required:
- normal text stream
- no usage event
- usage event at end
- disconnect mid-stream
- cancellation
- empty stream
- tool-call-only stream
- text then tool call
- multiple tool calls
- malformed tool arguments
- duplicate provider completion event
- stream timeout

Hard assertion:
exactly one terminal event.

---

# 29. Tool Security Test Matrix

Required:
- unknown tool
- disabled tool
- schema mismatch
- unexpected field
- wrong type
- missing required argument
- unauthorized state
- cross-tenant object ID
- duplicate idempotency key
- repeated handoff
- prompt injection requesting fabricated tools
- tool-like markup emitted as ordinary text

Tool-like text must not execute unless it enters the structured tool path.

---

# 30. Provider Conformance Tests

Every provider adapter must prove:
1. success maps to CanonicalAIResponse
2. errors map to CanonicalProviderError
3. secrets never appear in mapped errors
4. cancellation works
5. stream events are canonical
6. usage is normalized where available
7. tool calls are normalized where supported
8. unsupported capability fails explicitly
9. provider-specific types do not leak from runtime

---

# 31. Migration Plan

## Phase 0 — Characterize Current Behavior

Before refactor:
- identify current Mistral entrypoints
- capture regression fixtures
- record expected output shapes
- document quota behavior
- document error handling
- document tenant credential resolution
- document streaming behavior

No functional changes.

## Phase 1 — Canonical Contracts

Create canonical contract modules.

No provider behavior change.

Acceptance:
- existing Mistral tests pass
- public API behavior preserved

## Phase 2 — Mistral Adapter

Encapsulate current Mistral logic in providers/mistral/adapter.js.

Acceptance:
- business modules no longer import Mistral SDK directly
- existing behavior remains unchanged

## Phase 3 — Execution Runtime

All AI execution flows through ExecutionRuntime.execute().

Initial route:
Mistral / mistral-small-latest

No provider failover yet.

## Phase 4 — Central Error Mapping

Normalize provider errors and prohibit raw provider errors from reaching clients.

## Phase 5 — Credential Lease

Wrap existing encrypted credential resolution in a lease lifecycle.

Acceptance:
- tenant isolation tests pass
- release happens in finally path
- inflight count returns to zero after success/failure/cancellation

## Phase 6 — Retry Policy

Enable only bounded and safe same-tenant retries first:
- auth failure → alternate same-tenant credential
- rate limit → alternate same-tenant credential

## Phase 7 — Tool Runtime Foundation

Build registry, validation, authorization, idempotency, and fake tools.

Start with read-only tools:
- lookup_faq
- lookup_product
- lookup_price

Then controlled writes:
- update_lead_profile
- create_lead
- schedule_followup
- handoff_to_sales
- book_demo

## Phase 8 — Sales Agent Integration

Conversation State
→ Context Builder
→ CanonicalAIRequest
→ Gateway
→ CanonicalAIResponse / ToolCall
→ Action Policy
→ Lead / Handoff / Follow-up

## Phase 9 — Optional Multi-provider Failover

Only after actual need:
1. Mistral
2. Groq optional
3. OpenRouter optional

---

# 32. Rollback

Recommended flags:
- AI_GATEWAY_CANONICAL_CONTRACTS
- AI_GATEWAY_RUNTIME_V2
- AI_GATEWAY_CREDENTIAL_LEASE
- AI_GATEWAY_RETRY_V2
- AI_GATEWAY_TOOL_RUNTIME

If Ravina already has a feature-flag mechanism, reuse it.

Each migration phase should be independently reversible where practical.

---

# 33. Queueing

Do not introduce an unbounded in-memory AI queue.

For interactive DM replies:
- use bounded credential/provider concurrency
- optionally allow a short bounded wait
- otherwise return a typed busy/rate-limit result

Durable queues are for asynchronous work:
- scheduled follow-ups
- batch embeddings
- background enrichment

---

# 34. Conversation State Safety

Provider retries must not mutate conversation state until an accepted final result exists.

Read state
→ build request
→ provider execution/retry
→ accepted response
→ transactional state update

Tool side effects are separate and subject to idempotency.

---

# 35. Security Invariants

INV-SEC-01: secrets encrypted at rest.
INV-SEC-02: secrets never logged.
INV-SEC-03: tenantId mandatory for credential resolution.
INV-SEC-04: unknown caller fails closed.
INV-SEC-05: provider responses cannot directly invoke arbitrary code.
INV-SEC-06: tool arguments are validated.
INV-SEC-07: risky actions require policy authorization.
INV-SEC-08: human-owned conversations cannot be auto-sent by AI.
INV-SEC-09: retries cannot bypass tenant quota.
INV-SEC-10: provider failover cannot bypass tenant provider policy.

---

# 36. Definition of Done — Gateway Foundation

1. Canonical contracts exist.
2. Mistral adapter implements common provider contract.
3. Current AI features route through ExecutionRuntime.
4. Tenant credential isolation remains intact.
5. Central error mapping exists.
6. Telemetry records attempts without secrets.
7. Cancellation works.
8. Existing tests pass.
9. New conformance tests pass.
10. Provider SDK types do not leak into business modules.

---

# 37. Definition of Done — Retry/Failover

1. Explicit attempt state.
2. Maximum attempts enforced.
3. Same broken credential not retried repeatedly.
4. Alternate credential always same tenant.
5. 429/auth retry paths tested.
6. cancellation prevents retry.
7. emitted stream prevents unsafe replay.
8. side effects prevent unsafe retry.
9. retry telemetry exists.
10. failover is configuration-driven.

---

# 38. Definition of Done — Tool Runtime

1. explicit registry
2. unknown tools rejected
3. schema validation enforced
4. authorization enforced
5. idempotency for side effects
6. safe result normalization
7. no sensitive arguments in logs
8. cross-tenant object access denied
9. human handoff idempotent
10. tool-like plain text cannot execute

---

# 39. Codex Work Packages

## WP-01 — Discovery and Regression Lock

Tasks:
- inspect current Ravina AI Gateway paths
- identify all Mistral imports and calls
- identify credential resolution and encryption/decryption
- identify quota handling
- identify provider routing
- identify Direct Reply, Form Builder, Embeddings, Intent Detection, and Lead Qualification integration points
- identify streaming behavior
- identify current tests
- add regression tests where behavior is unprotected
- produce docs/AI_GATEWAY_CURRENT_IMPLEMENTATION_MAP.md

No production behavior change.

## WP-02 — Canonical Contracts

- create contract modules
- add validation
- add unit tests
- document boundaries

## WP-03 — Mistral Adapter

- encapsulate current Mistral client logic
- implement provider contract
- normalize response and error behavior
- add conformance tests

## WP-04 — Execution Runtime

- ExecutionContext
- AttemptTracker
- runtime orchestration
- telemetry hooks
- cancellation
- preserve output behavior

## WP-05 — Credential Lease

- wrap current encrypted credential lookup
- enforce tenant match
- inflight tracking
- release in finally
- concurrency tests
- tenant isolation tests

## WP-06 — Error and Retry Policy

- canonical errors
- deterministic retry decision
- bounded same-tenant alternate credential retry
- retry matrix tests

## WP-07 — Streaming Runtime

- canonical stream events
- exactly one terminal event
- cancellation
- tool-event isolation

## WP-08 — Tool Runtime Foundation

- registry
- validation
- authorization
- idempotency interface
- fake tools
- no risky business writes yet

## WP-09 — Sales Tools

Recommended sequence:
1. lookup_faq
2. lookup_product
3. lookup_price
4. update_lead_profile
5. create_lead
6. schedule_followup
7. handoff_to_sales
8. book_demo

---

# 40. Codex Operating Rules

Codex must:
1. inspect before modifying
2. preserve existing project conventions
3. avoid unrelated refactors
4. keep work packages independently testable
5. run existing tests after each package
6. add tests for every new invariant
7. never commit secrets
8. never weaken tenant isolation
9. never enable cross-tenant fallback
10. never replace deterministic lead scoring with LLM scoring
11. never enable AI auto-send during human ownership
12. report contradictions with security invariants before proceeding

---

# 41. Non-goals

This work does not include:
- reverse-engineering DeepSeek Web
- deploying DS2API inside Ravina
- copying DS2API parser code
- microservice decomposition
- Kubernetes
- distributed queues for interactive chat
- speculative provider integrations
- unrestricted autonomous tool execution
- LLM-controlled lead scoring
- changing product roadmap priorities

---

# 42. Product Priority Guardrail

Gateway refactoring is subordinate to Ravina's product roadmap:

Conversation State Engine
→ Lead Profile Foundation
→ Lead Qualification
→ Lead Scoring
→ Next Best Action
→ Human Handoff
→ Operator Workspace

No gateway refactor should delay the next end-to-end Sales Agent milestone unless it removes a direct blocker.

---

# 43. First Codex Task

Codex must execute WP-01 only.

Prompt:

You are working on the Ravina AI Gateway codebase.

Read docs/RAVINA_AI_GATEWAY_CLEANROOM_SPEC.md as the implementation contract.

Execute WP-01 — Discovery and Regression Lock only.

Goals:
1. Locate every current AI Gateway entrypoint.
2. Locate all Mistral SDK/client imports and calls.
3. Locate tenant credential resolution and encryption/decryption paths.
4. Locate quota/rate-limit handling.
5. Locate current provider routing logic.
6. Locate Direct Reply, Form Builder, Embeddings, Intent Detection, and Lead Qualification integrations.
7. Identify current streaming behavior.
8. Identify current tests protecting AI Gateway behavior.
9. Add regression tests where current behavior is unprotected.
10. Produce docs/AI_GATEWAY_CURRENT_IMPLEMENTATION_MAP.md.

Constraints:
- Do not redesign production code yet.
- Do not introduce new providers.
- Do not change database schema.
- Do not change public API behavior.
- Do not weaken tenant isolation.
- Do not copy code from DS2API.
- Do not implement WP-02 or later work packages.
- Run the relevant existing test suite and report results.

The implementation map must include:
- file paths
- responsibilities
- call flow
- credential flow
- quota flow
- error flow
- streaming flow
- known technical debt
- test coverage gaps
- exact recommended insertion points for WP-02 and WP-03

Return:
1. files inspected
2. files changed
3. tests added
4. tests run and results
5. risks found
6. recommended next step

Do not proceed to WP-02 until WP-01 results are reviewed.
