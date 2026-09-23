# 02 · Technology selection: <product-name>

Technology is chosen from the requirements, not the other way round. Each decision below is evaluated with the same weighted method and recorded as an ADR in [adr/](adr/).

## 1. Decisions to make

| Decision | Driven by requirements | ADR |
|---|---|---|
| Language and runtime | NFR-, SR- | [ADR-](adr/) |
| Web / API framework | | |
| Database | NFR-, data requirements | |
| Data access and migrations | | |
| Authentication approach | SR- | |
| Test tooling (unit, integration, E2E, load) | NFR- | |
| Container base image | SR-, NFR-005 | |

## 2. Knock-out criteria

A candidate that fails any of these is eliminated before scoring:

- OSI-approved license compatible with Apache-2.0
- Actively maintained: a release in the last 6 months and security advisories handled publicly
- Supports `linux/amd64` and `linux/arm64`
- Runs within the $0 budget constraint

## 3. Weighted criteria

Default weights — adjust per decision and justify changes in the ADR.

| Criterion | Weight | What a 5 looks like |
|---|---|---|
| Requirement fit | 25% | Meets every linked NFR without workarounds |
| Security | 20% | Secure defaults, good CVE track record, signed releases, small attack surface |
| Operability | 15% | Observability built in, small image, predictable memory, easy upgrades |
| Ecosystem and maturity | 15% | Stable APIs, rich libraries, strong documentation, large community |
| Hiring market relevance | 10% | Widely used in DevSecOps / platform teams |
| Learning curve | 10% | Productive within the project timeline |
| Licensing and cost | 5% | Permissive license, no paid tiers required |

Scores are 1 (poor) to 5 (excellent). Weighted total = Σ(weight × score).

## 4. Evaluations

### 4.1 <Decision name>

| Criterion | Weight | <Candidate A> | <Candidate B> | <Candidate C> |
|---|---|---|---|---|
| Requirement fit | 25% | | | |
| Security | 20% | | | |
| Operability | 15% | | | |
| Ecosystem and maturity | 15% | | | |
| Hiring market relevance | 10% | | | |
| Learning curve | 10% | | | |
| Licensing and cost | 5% | | | |
| **Weighted total** | | | | |

**Evidence:** <links to benchmarks, CVE history, docs, prototypes>
**Decision:** <chosen candidate> — see [ADR-NNNN](adr/).

## Gate checklist

- [ ] Every decision in section 1 has an ADR
- [ ] Every ADR links the requirement IDs that drove it
- [ ] Every candidate passed the knock-out criteria or was documented as eliminated
- [ ] Scores are backed by evidence, not preference
