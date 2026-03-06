---
name: principal-engineer
description: Orchestrates parallel principal-engineer-level code review and automated remediation across an entire repository. Spawns 4 reviewer agents covering architecture, security, performance, and quality, then up to 5 implementation agents to fix all critical/high issues with test-backed verification. Use when the user asks for a comprehensive code review, deep audit, full codebase review-and-fix, or principal-engineer review.
---

# Principal Engineer Review & Remediation

Two-phase workflow: **Review** (4 parallel agents) then **Fix** (up to 5 parallel agents).
Every finding is severity-classified; every fix is test-validated.

---

## Phase 0: Stack Detection

Before launching reviewers, identify the project's tech stack:

1. Read dependency manifests (`package.json`, `go.mod`, `requirements.txt`, `Cargo.toml`, `pom.xml`, etc.).
2. Scan for framework config files, CI definitions, Dockerfiles.
3. Note the primary language(s), frameworks, build system, and test runner.
4. Pass this context to every agent prompt via `{STACK_CONTEXT}`.

---

## Phase 1: Review

Launch **4 Task agents in a single message** (parallel). Each uses `subagent_type: "explore"` with thoroughness `"very thorough"`.

### Agent Assignments

| Agent | Domains |
|-------|---------|
| R1 — Architecture | System design, module boundaries, API design, contract stability, data modeling, state management, dependency graph |
| R2 — Security | Application security, dependency vulnerabilities, infrastructure risks, supply chain, secrets handling, input validation, auth/authz |
| R3 — Performance | CPU/memory/IO hotspots, concurrency correctness, scalability limits, connection pooling, caching, resource leaks |
| R4 — Quality | Code quality, maintainability, test coverage & determinism, edge-case gaps, observability (logging, metrics, tracing), CI/CD safety, error handling |

### Agent Prompt Template

Replace `{DOMAIN}`, `{AREAS}`, `{FOCUS_QUESTIONS}`, and `{STACK_CONTEXT}` per agent.

```
You are a principal engineer (20+ years experience) conducting a production-grade review.

TECH STACK: {STACK_CONTEXT}
SCOPE: {DOMAIN}
AREAS: {AREAS}

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

## Recommendations
[Prioritized list of improvements]
```

### Focus Questions by Agent

**R1 — Architecture:**
- Are module boundaries clean? Any circular dependencies?
- Is the API surface consistent and versioned?
- Are data models normalized appropriately? Any schema drift risk?
- Is the dependency graph minimal and justified?
- Are there architectural single points of failure?

**R2 — Security:**
- Any injection vectors (SQL, NoSQL, command, header)?
- Are secrets hardcoded or logged?
- Is auth/authz enforced consistently on all mutation paths?
- Are dependencies pinned? Any known CVEs?
- Is input validation comprehensive (size, type, encoding)?
- Any SSRF, path traversal, or CORS misconfiguration?

**R3 — Performance:**
- Any unbounded allocations, goroutine leaks, thread pool exhaustion, or missing context cancellation?
- Are database/storage calls batched or parallelized where possible?
- Is there unnecessary serialization or copying on hot paths?
- Are there race conditions or data races?
- Any missing connection pooling or resource limits?
- Are timeouts set on all external calls?

**R4 — Quality:**
- What is the effective test coverage? Any untested critical paths?
- Are tests deterministic (no time-dependent, no external deps)?
- Is error handling consistent (no swallowed errors)?
- Are logs structured with correlation IDs?
- Is the CI pipeline testing edge cases and fuzzing?
- Any dead code, unused exports, or stale dependencies?

---

## Phase 1.5: Consolidation

After all 4 agents return:

1. Collect all findings into a single prioritized list.
2. Deduplicate overlapping findings (keep the more detailed version).
3. Create a TodoWrite checklist grouping issues by severity then by domain.
4. Present the consolidated report to the user before proceeding.
5. **Ask the user for confirmation** before starting Phase 2.

---

## Phase 2: Fix

Launch **up to 5 Task agents in a single message** (parallel). Use `subagent_type: "generalPurpose"`. Only launch agents whose category has findings.

| Agent | Responsibility |
|-------|---------------|
| F1 — Critical Fixes | All critical and high-severity issues across domains |
| F2 — Architecture | Refactor architectural weaknesses, improve module boundaries |
| F3 — Security | Eliminate security vulnerabilities, harden auth/input validation |
| F4 — Performance | Optimize hotspots with measurable benchmarks |
| F5 — Reliability | Strengthen error handling, add missing tests, reduce tech debt |

### Agent Prompt Template

```
You are a principal engineer implementing fixes in a production codebase.

TECH STACK: {STACK_CONTEXT}
ASSIGNMENT: {RESPONSIBILITY}

FINDINGS TO ADDRESS:
{PASTE_RELEVANT_FINDINGS}

MANDATORY CONSTRAINTS:
1. No change may break existing business logic.
2. All behavior must remain functionally equivalent unless explicitly justified.
3. Before modifying any code, verify existing test coverage for that code path.
4. If coverage is insufficient, ADD tests first that lock current behavior.
5. After each fix, run tests to confirm behavioral parity.
6. For performance fixes, capture before/after measurements.
7. Every modification must be backed by a test proving correctness.

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
- Before/after metrics (for performance fixes)

## Test Results
- Tests added: N
- Tests modified: N
- Full suite result: PASS/FAIL

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
   - Performance deltas (if applicable)
   - Any deferred items with justification
4. Present the report to the user.

---

## Execution Checklist

Track with TodoWrite:

```
- [ ] Detect tech stack (Phase 0)
- [ ] Launch 4 reviewer agents (parallel)
- [ ] Collect and consolidate findings
- [ ] Present review report to user
- [ ] Get user confirmation to proceed with fixes
- [ ] Launch fix agents (parallel, up to 5)
- [ ] Run full test suite after fixes
- [ ] Present final validation report
```

---

## Constraints

- Cap each reviewer agent at `very thorough` exploration.
- Implementation agents must not make speculative changes — only address documented findings.
- If the repository has no test infrastructure, the first fix agent must set it up before other agents proceed (launch F1 first, then F2–F5 after it completes).
