# 07 · Delivery plan: <product-name>

Work is tracked as GitHub issues grouped into milestones. This document is the plan; the issues are the source of truth for status.

## 1. Milestones

| Milestone | Scope (requirement IDs) | Exit criteria | Target date | Status |
|---|---|---|---|---|
| M0 · Walking skeleton | Health endpoints, empty schema, full pipeline | Phase 6 gate passed | | ⬜ |
| M1 · <name> | FR-, FR- | | | ⬜ |
| M2 · <name> | | | | ⬜ |
| v1.0.0 | All Must-have FRs | Phase 8 gate passed | | ⬜ |

## 2. Release plan

- Versioning: Semantic Versioning. `0.x` releases may break compatibility; `1.0.0` freezes the public API.
- A release is cut by pushing a `vX.Y.Z` tag on a commit that already passed CI on `main`.
- Every release is re-verified in a clean cluster before any artifact is published.

## 3. Risks and issues log

| Date | Risk / issue | Impact | Action | Status |
|---|---|---|---|---|
| | | | | |

## Gate checklist

- [ ] Every milestone delivered or explicitly re-planned
- [ ] Every merged PR met the Definition of Done
