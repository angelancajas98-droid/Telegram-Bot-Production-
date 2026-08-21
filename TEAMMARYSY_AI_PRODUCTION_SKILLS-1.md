# AI Skills — TeamMarySy Production-Grade Deployment

This document captures the practical engineering skills/guidance used to analyze and rebuild the TeamMarySy Telegram bot. It is intended to be reusable with another AI coding agent.

## 1. Production Architecture Review

Role:
Act as a senior cloud/platform architect reviewing an existing application for production deployment.

Requirements:
- Inspect the actual repository/code, not documentation alone.
- Identify discrepancies between claimed features and implemented behavior.
- Separate infrastructure readiness from business-feature completeness.
- Evaluate request flow, state management, concurrency, failure modes, security, deployment, observability, and recovery.
- Prefer explicit source-of-truth documents over assumptions.
- Do not call a feature production-ready when it is only a UI stub or placeholder.

Primary output:
- Architecture assessment
- P0/P1/P2 risks
- Required changes
- Production-readiness decision

## 2. Cloudflare Workers Production Engineering

Role:
Design Cloudflare Workers systems using the correct primitive for each workload.

Rules:
- Use Workers for stateless request handling and API orchestration.
- Use Workers KV for eventually consistent, read-heavy configuration and lightweight metadata.
- Do NOT use KV as a transactional database, distributed lock, strict atomic counter, or queue.
- Use Durable Objects for strong coordination, atomic state transitions, per-entity serialization, idempotency, rate limiting, and leases/locks.
- Use Cloudflare Queues for asynchronous/deferred work, retries, and dead-letter handling.
- Keep webhook/request handlers fast.
- Move heavy or retryable outbound operations out of the synchronous webhook path.
- Treat every external API call as failure-prone.
- Make configuration and deployment declarations authoritative and internally consistent.

## 3. Telegram Bot Reliability Engineering

Role:
Build a production Telegram Bot API integration.

Requirements:
- Validate `X-Telegram-Bot-Api-Secret-Token`.
- Use webhook `update_id` for idempotency/deduplication.
- Authenticate and authorize the actor before feature execution.
- Separate authentication, authorization, routing, and feature logic.
- Always acknowledge callback queries promptly.
- Implement Telegram API retry/backoff for transient failures:
  - HTTP 429
  - HTTP 5xx
  - network/timeouts
- Respect `Retry-After` where supplied.
- Design outbound sends to be retry-safe.
- Avoid duplicate broadcasts/messages on job retries.
- Never log bot tokens, webhook secrets, message content, or unnecessary PII.

## 4. Distributed-State and Concurrency Review

Role:
Analyze every read-modify-write flow for races.

For each state transition ask:
1. Can two requests execute simultaneously?
2. Is the operation atomic?
3. Is eventual consistency acceptable?
4. Can the action be retried?
5. Can the same action be executed twice?
6. What happens if the Worker crashes between state write and external API call?

Typical race conditions to look for:
- KV `get → increment → put`
- KV `get → create → put`
- KV-based job locks
- ticket uniqueness
- sequential IDs
- scheduled-job claiming
- duplicate webhook processing
- repeated external API sends

Preferred mitigation:
- Durable Object state and serialized execution
- idempotency keys
- transactional state transitions where required
- leases with expiry
- queue-based delivery for asynchronous operations

## 5. Security Engineering

Role:
Review the system as a production security boundary.

Requirements:
- Secrets must exist only in Cloudflare secret bindings/secure CI variables.
- Never commit bot tokens or production credentials.
- Validate webhook authenticity before processing.
- Centralize owner/admin/role evaluation.
- Apply least privilege per command/feature.
- Do not infer administrative privilege from the identity of an end user in an event such as a join request.
- Treat user-supplied callback data, command arguments, chat IDs, and identifiers as untrusted input.
- Avoid logging message bodies and sensitive personal data.
- Define staging and production as separate environments with separate credentials.
- Provide a credential rotation procedure.

## 6. Deployment-as-Code Engineering

Role:
Make the repository the authoritative deployment source.

Requirements:
- One authoritative Wrangler configuration.
- Explicit bindings for:
  - KV
  - Durable Objects
  - Queues
- Explicit environment configuration for staging/production.
- CI should build and validate before deployment.
- Production deployment should be protected/manual unless the organization explicitly requires automatic production release.
- Smoke tests must test real endpoints and real deployment assumptions.
- Avoid dashboard-only configuration that is not represented in source control.
- Keep deployment documentation synchronized with code/config.

## 7. Observability and Operations

Role:
Design minimal but sufficient production observability.

Log structured metadata such as:
- timestamp
- update type
- handler
- command/callback action
- non-sensitive actor/chat identifiers where operationally justified
- result
- duration
- error classification

Do NOT log:
- Telegram message contents
- bot tokens
- webhook secrets
- unnecessary PII

Track:
- webhook request latency
- processing errors
- Telegram API 429s
- Telegram API 5xxs
- queue failures
- DLQ count
- duplicate updates
- job retries
- scheduled job failures

Every critical operational failure should have a way to diagnose it without inspecting user conversation content.

## 8. Testing Strategy

Role:
Test production behavior, not just source structure.

Minimum tests:
- webhook secret rejection
- malformed update rejection
- update idempotency
- command routing
- authorization
- callback acknowledgment
- rate limiting
- Telegram 429 retry behavior
- Telegram 5xx retry behavior
- support-ticket uniqueness
- job claiming
- queue failure/retry handling
- scheduled-job idempotency
- health endpoint
- environment/config validation

Tests should include concurrency-oriented cases where practical.

A green test suite is not sufficient evidence of production readiness unless it covers the primary failure modes.

## 9. Source-of-Truth Discipline

Maintain explicit authoritative documents:

`README.md`
- system overview
- local development
- project structure

`ARCHITECTURE.md`
- runtime architecture
- state model
- consistency model
- concurrency model
- failure model

`DEPLOYMENT.md`
- exact deployment procedure
- required bindings
- secrets
- webhook setup
- rollback
- smoke tests

`OPERATIONS.md`
- incident handling
- retries
- queue/DLQ handling
- credential rotation
- maintenance

`FEATURE_STATUS.md`
- completed
- partial
- placeholder
- not implemented

`wrangler.jsonc`
- authoritative infrastructure configuration

The documentation must never claim capabilities that the code does not implement.

## 10. AI Coding-Agent Operating Procedure

When given an existing repository:

1. Inventory the repository.
2. Identify the actual runtime entry point.
3. Read configuration and deployment files.
4. Trace every externally visible feature to its implementation.
5. Identify stubs/placeholders.
6. Build a threat model.
7. Identify concurrency/state races.
8. Identify retry/idempotency requirements.
9. Design the smallest architecture that correctly solves those problems.
10. Implement infrastructure changes.
11. Implement reliability/security changes.
12. Add tests for failure modes.
13. Run formatting, linting, type checking, and tests.
14. Re-read documentation against the final code.
15. Remove contradictions.
16. Produce a production-readiness report with remaining gaps.
17. Never state that deployment or an external action succeeded unless it was actually verified.

## 11. Production Readiness Gate

Do not classify a system as production-ready unless all of the following are true:

- Secrets are externally managed.
- Webhook authentication is enforced.
- Duplicate updates are safely handled.
- Strictly consistent operations do not rely on KV.
- Retryable external calls have a retry policy.
- Heavy work is asynchronous where appropriate.
- Scheduled jobs cannot be executed twice due to lock races.
- Authorization is centralized and tested.
- CI validates the actual production build.
- Health/smoke checks target real endpoints.
- Rollback procedure exists.
- Operational documentation matches the code.
- Unimplemented features are explicitly labeled.
- Critical failure paths have automated tests.

## 12. Recommended AI Output Format

For architecture work, report:

1. Executive assessment
2. Current architecture
3. Critical findings
4. Security findings
5. Consistency/concurrency findings
6. Reliability findings
7. Deployment findings
8. Required remediation
9. Files changed
10. Validation performed
11. Remaining limitations
12. Final production-readiness rating

Never hide uncertainty. Distinguish:
- verified
- inferred
- recommended
- not implemented
