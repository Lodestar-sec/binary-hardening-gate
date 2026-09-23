# Project documentation

Every Lodestar Security product follows the same phase-gated lifecycle. A phase is complete only when its gate criteria are met; the next phase does not start before that.

Technology choices (language, framework, database) are made in **phase 2**, derived from the requirements in phase 1 — never before.

| # | Phase | Document | Gate: complete when | Status |
|---|---|---|---|---|
| 0 | Inception | [00-charter.md](00-charter.md) + [research plan](research/phase-0-plan.md) | Problem, users, scope, non-goals, and success metrics are explicit and measurable | 🟨 |
| 1 | Requirements | [01-requirements.md](01-requirements.md) | Every requirement has an ID, is testable, and has acceptance criteria; security requirements mapped to OWASP ASVS L2 | ⬜ |
| 2 | Technology selection | [02-tech-selection.md](02-tech-selection.md) + [ADRs](adr/) | Every choice has an ADR with a weighted evaluation and traces back to requirement IDs | ⬜ |
| 3 | Architecture | [03-architecture.md](03-architecture.md) | C4 levels 1–3 drawn; every NFR mapped to an architectural mechanism; API contract drafted | ⬜ |
| 4 | Data design | [04-data-model.md](04-data-model.md) | Every core query has a supporting index; every business rule is enforced in the database where possible; RPO/RTO defined | ⬜ |
| 5 | Threat model | [05-threat-model.md](05-threat-model.md) | Every High threat is mitigated or formally accepted with an owner and review date | ⬜ |
| 6 | Engineering standards | [06-engineering-standards.md](06-engineering-standards.md) | Walking skeleton: the first commit passes the full CI/CD pipeline, including the ephemeral deployment | ⬜ |
| 7 | Delivery | [07-delivery-plan.md](07-delivery-plan.md) | Milestones delivered; every merged PR met the Definition of Done | ⬜ |
| 8 | Verification | [08-verification.md](08-verification.md) | Every NFR and security requirement verified with linked evidence | ⬜ |
| 9 | Operations | [09-runbook.md](09-runbook.md) | Release verified in a clean cluster; runbook covers every alert; rollback tested | ⬜ |

Status legend: ⬜ not started · 🟨 in progress · ✅ gate passed (link the PR that passed it).

## Conventions

- Gates are passed through a pull request that updates the status column and links the evidence.
- Documents are living: update them when reality changes, and record significant changes as ADRs.
- Requirement IDs (`FR-`, `NFR-`, `SR-`, `AC-`) are stable and referenced from code, tests, and ADRs for traceability.
