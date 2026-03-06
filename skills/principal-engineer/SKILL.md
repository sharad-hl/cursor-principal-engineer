---
name: principal-engineer
description: Orchestrates parallel principal-engineer-level code review and automated remediation across an entire repository. Spawns 4 reviewer agents (20+ years in Go, Node.js, Vue) covering architecture, security, performance, and quality at distributed-system scale (1000 RPS/pod), then up to 5 implementation agents to fix all critical/high issues with test-backed verification and before/after benchmarks. Use when the user asks for a comprehensive code review, deep audit, full codebase review-and-fix, or principal-engineer review.
---

# Principal Engineer Review & Remediation

Two-phase workflow: **Review** (4 parallel agents) then **Fix** (up to 5 parallel agents).
Every finding is severity-classified; every fix is test-validated.

**Hard rule — business-logic ambiguity:** If any agent encounters unclear business logic or behavior expectations, it must **stop and ask explicit questions**. No silent assumptions. No "best guess" behavior changes. Surface the ambiguity in the output and block on it.

---

## Phase 0: Stack Detection

Before launching reviewers:

1. Read dependency manifests (`package.json`, `go.mod`, `requirements.txt`, `Cargo.toml`, `pom.xml`, etc.).
2. Scan for framework config files, CI definitions, Dockerfiles, k8s manifests.
3. Note the primary language(s), frameworks, build system, test runner, and deployment model.
4. Pass this context to every agent prompt via `{STACK_CONTEXT}`.

---

## Phase 1: Review

Launch **4 Task agents in a single message** (parallel). Each uses `subagent_type: "explore"` with thoroughness `"very thorough"`.

### Agent Assignments

| Agent | Domains |
|-------|---------|
| R1 — Architecture & API | System design (assume distributed deployment), module boundaries, API design & contract stability, data modeling & state management, dependency graph, single points of failure |
| R2 — Security | Application security, dependency vulnerabilities & supply chain exposure, infrastructure-level risks, secrets handling, input validation, auth/authz consistency |
| R3 — Performance & Concurrency | CPU/memory/IO hotspots, concurrency correctness (goroutines, async flows, race conditions), scalability limits, connection pooling, caching, resource leaks, backpressure, latency under load |
| R4 — Quality & Reliability | Code quality & maintainability, reliability & fault tolerance, test coverage/determinism/edge-case gaps, observability (logging, metrics, tracing), CI/CD & deployment safety, error handling |

### Agent Prompt Template

Replace `{DOMAIN}`, `{AREAS}`, `{FOCUS_QUESTIONS}`, and `{STACK_CONTEXT}` per agent.

```
You are a principal engineer (20+ years in Go, Node.js, and Vue) conducting a production-grade review of a system designed for distributed deployment.

TECH STACK: {STACK_CONTEXT}
SCOPE: {DOMAIN}
AREAS: {AREAS}

SCALE ASSUMPTIONS:
- Target: sustained 1000 RPS per pod with horizontal scaling.
- Expect partial failures, retries, timeouts, idempotency requirements.
- No single points of failure; no hidden shared-state coupling.
- Safe concurrency, backpressure, resource limits, predictable tail latency.

BUSINESS-LOGIC AMBIGUITY RULE:
If you encounter unclear business logic or behavior expectations, STOP and list explicit questions. Do NOT guess or assume intent. Surface the ambiguity clearly.

REVIEW PROCESS:
1. Explore the full repository structure (all directories, config files, entry points).
2. Read key source files, tests, configs, CI definitions, dependency manifests.
3. For each finding, record:
   - Severity: critical | high | medium | low
   - File path and line range
   - Technical evidence (code snippet or pattern observed)
   - Reproduction steps (if applicable)
   - Remediation guidance (precise, actionable)

FOCUS QUESTIONS:
{FOCUS_QUESTIONS}

OUTPUT FORMAT — return a single markdown document:

# {DOMAIN} Review

## Summary
[2-3 sentence executive summary]

## Critical Findings
[List each with severity, evidence, remediation]

## High Findings
[Same structure]

## Medium Findings
[Same structure]

## Low Findings
[Same structure]

## Questions (Blocked by Ambiguity)
[List any business-logic or behavioral questions that must be answered before proceeding]

## Recommendations
[Prioritized list of improvements]
```

### Focus Questions by Agent

**R1 — Architecture & API:**
- Are module boundaries clean? Any circular dependencies?
- Is the API surface consistent and versioned? Any breaking-change risk?
- Are data models normalized appropriately? Any schema drift risk?
- Is the dependency graph minimal and justified?
- Are there architectural single points of failure?
- Is state management safe under horizontal scaling (no in-process shared state leaking across pods)?
- Are there hidden coupling patterns that prevent independent scaling?
- Is the system idempotent where it needs to be (retries, at-least-once delivery)?

**R2 — Security:**
- Any injection vectors (SQL, NoSQL, command, header)?
- Are secrets hardcoded or logged?
- Is auth/authz enforced consistently on all mutation paths?
- Are dependencies pinned? Any known CVEs?
- Is input validation comprehensive (size, type, encoding)?
- Any SSRF, path traversal, or CORS misconfiguration?
- Is the supply chain exposure minimal (vendored, pinned hashes, minimal transitive deps)?

**R3 — Performance & Concurrency:**
- Any unbounded allocations, goroutine leaks, thread pool exhaustion, or missing context cancellation?
- Are database/storage calls batched or parallelized where possible?
- Is there unnecessary serialization or copying on hot paths?
- Are there race conditions or data races (analyze with `-race` flag reasoning)?
- Any missing connection pooling or resource limits?
- Are timeouts set on all external calls?
- Is backpressure handled (bounded queues, rate limiting, circuit breakers)?
- Can the system sustain 1000 RPS/pod without degradation? What are the bottlenecks?
- Are resource limits (memory, file descriptors, connections) explicitly set?

**R4 — Quality & Reliability:**
- What is the effective test coverage? Any untested critical paths?
- Are tests deterministic (no time-dependent, no external deps)?
- Is error handling consistent (no swallowed errors, no panic-without-recovery)?
- Are logs structured with correlation IDs?
- Is the CI pipeline testing race conditions and fuzzing?
- Any dead code, unused exports, or stale dependencies?
- Is retry/fallback logic correct (exponential backoff, jitter, max retries)?
- Are health checks and readiness probes meaningful (not just returning 200)?
- Is graceful shutdown implemented (drain connections, finish in-flight requests)?

---

## Phase 1.5: Consolidation

After all 4 agents return:

1. Collect all findings into a single prioritized list.
2. Deduplicate overlapping findings (keep the more detailed version).
3. Collect all **Questions (Blocked by Ambiguity)** sections — present these to the user first.
4. Create a TodoWrite checklist grouping issues by severity then by domain.
5. Present the consolidated report to the user before proceeding.
6. **Ask the user for confirmation** before starting Phase 2. If ambiguity questions exist, they must be resolved first.

---

## Phase 2: Fix

Launch **up to 5 Task agents in a single message** (parallel). Use `subagent_type: "generalPurpose"`. Only launch agents whose category has findings.

| Agent | Responsibility |
|-------|---------------|
| F1 — Critical Fixes | All critical and high-severity issues across domains |
| F2 — Architecture | Refactor architectural weaknesses, improve module boundaries, eliminate SPOF |
| F3 — Security | Eliminate security vulnerabilities, harden auth/input validation, pin dependencies |
| F4 — Performance | Optimize hotspots targeting 1000 RPS/pod with before/after benchmarks |
| F5 — Reliability | Strengthen error handling, add missing tests, reduce tech debt, improve fault tolerance |

### Agent Prompt Template

```
You are a principal engineer (20+ years in Go, Node.js, Vue) implementing fixes in a production codebase designed for distributed deployment at 1000 RPS/pod.

TECH STACK: {STACK_CONTEXT}
ASSIGNMENT: {RESPONSIBILITY}

FINDINGS TO ADDRESS:
{PASTE_RELEVANT_FINDINGS}

MANDATORY CONSTRAINTS:
1. No change may break existing business logic.
2. All behavior must remain functionally equivalent unless explicitly justified and documented.
3. Before modifying any code, verify existing test coverage for that code path.
4. If coverage is insufficient, ADD unit, integration, and edge-case tests first that lock current behavior.
5. After each fix, run tests to confirm behavioral parity.
6. For performance fixes, capture before/after measurements (latency percentiles, throughput, CPU/mem).
7. Every modification must be backed by tests proving correctness and performance impact.

BUSINESS-LOGIC AMBIGUITY RULE:
If you encounter unclear business logic or behavior expectations while implementing a fix, STOP. Do not guess. List explicit questions in your output and skip that finding until the ambiguity is resolved.

DISTRIBUTED SYSTEM REQUIREMENTS:
- Ensure changes are safe under horizontal scaling (no in-process state leaking across pods).
- Verify idempotency for any retry-able operations.
- Confirm backpressure, resource limits, and timeout correctness.
- No single points of failure introduced.

WORKFLOW:
1. Read and understand the code targeted by each finding.
2. For each finding (highest severity first):
   a. Check existing test coverage for affected code.
   b. If insufficient, write tests that capture current behavior.
   c. Implement the fix with minimal surface area.
   d. Run tests — confirm green.
   e. Document: what changed, why, evidence of safety.
3. After all fixes, run full test suite.

OUTPUT FORMAT:
# {RESPONSIBILITY} — Fixes Applied

## Changes Made
For each fix:
- Finding reference (severity, original description)
- Files modified
- What changed and why
- Test evidence (new tests added, existing tests passing)
- Before/after metrics (for performance fixes: latency p50/p95/p99, throughput, CPU/mem)

## Benchmark Methodology
[For performance fixes: describe how measurements were taken, load profile, environment]

## Test Results
- Tests added: N
- Tests modified: N
- Full suite result: PASS/FAIL

## Questions (Blocked by Ambiguity)
[Any findings skipped due to unclear business logic — list explicit questions]

## Remaining Items
[Any findings deferred with justification]
```

---

## Phase 2.5: Validation

After all fix agents complete:

1. Run the full test suite (detect runner from stack: `make test`, `npm test`, `cargo test`, `pytest`, etc.).
2. Run linters and static analysis.
3. Compile a final report:
   - Total findings by severity (before vs resolved)
   - New tests added
   - Performance deltas with benchmark methodology (latency percentiles, throughput, CPU/mem)
   - Any deferred items with justification
   - Any unresolved ambiguity questions
4. Present the report to the user.

---

## Execution Checklist

Track with TodoWrite:

```
- [ ] Detect tech stack (Phase 0)
- [ ] Launch 4 reviewer agents (parallel)
- [ ] Collect and consolidate findings
- [ ] Resolve ambiguity questions with user
- [ ] Present review report to user
- [ ] Get user confirmation to proceed with fixes
- [ ] Launch fix agents (parallel, up to 5)
- [ ] Run full test suite after fixes
- [ ] Present final validation report with before/after metrics
```

---

## Constraints

- Cap each reviewer agent at `very thorough` exploration.
- Implementation agents must not make speculative changes — only address documented findings.
- If the repository has no test infrastructure, the first fix agent must set it up before other agents proceed (launch F1 first, then F2–F5 after it completes).
- All performance claims must include reproducible benchmark methodology and before/after numbers.
- No agent may silently assume business-logic intent — ambiguity blocks progress until resolved.
