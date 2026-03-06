# Principal Engineer — Cursor Plugin

Multi-agent code review and automated remediation at principal-engineer level.

## What it does

Runs a two-phase workflow on any repository:

**Phase 1 — Review** (4 parallel agents):

| Agent | Coverage |
|-------|----------|
| Architecture | System design, API contracts, data modeling, dependency graph |
| Security | Vulnerabilities, auth/authz, supply chain, secrets, input validation |
| Performance | Hotspots, concurrency, race conditions, resource management |
| Quality | Test coverage, observability, error handling, CI/CD, dead code |

**Phase 2 — Fix** (up to 5 parallel agents):

| Agent | Scope |
|-------|-------|
| Critical Fixes | All critical and high-severity issues |
| Architecture | Module boundaries, structural refactoring |
| Security | Vulnerability elimination, hardening |
| Performance | Optimization with before/after benchmarks |
| Reliability | Error handling, test coverage, tech debt |

## How it works

1. Auto-detects your tech stack from dependency manifests and config files.
2. Launches 4 reviewer agents that explore the entire codebase in parallel.
3. Consolidates and deduplicates findings by severity (critical → low).
4. Presents a unified report and waits for your confirmation.
5. Launches up to 5 fix agents that address documented findings.
6. Every fix is test-validated — no behavior regressions.
7. Produces a final report with before/after metrics.

## Constraints enforced

- No business logic breakage — behavioral parity is verified by tests.
- Tests are added before refactoring if coverage is insufficient.
- Implementation agents only address documented findings, no speculative changes.
- Performance fixes include measurable before/after evidence.

## Usage

After installing the plugin, invoke the skill in Cursor chat:

```
/principal-engineer
```

Or describe what you want and the agent will pick it up automatically:
- "Run a comprehensive code review"
- "Deep audit this repository"
- "Principal engineer review and fix"

## Install

Install from the [Cursor Marketplace](https://cursor.com/marketplace) or add manually:

```bash
# Clone into your Cursor plugins directory
git clone https://github.com/sharad/cursor-principal-engineer.git
```

## License

MIT
