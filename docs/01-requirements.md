# 01 · Requirements: <product-name>

| | |
|---|---|
| **Status** | Draft · In review · Approved |
| **Last updated** | YYYY-MM-DD |

## 1. Glossary

| Term | Definition |
|---|---|
| | |

## 2. Functional requirements

<!-- One row per user story. Priority uses MoSCoW: Must / Should / Could / Won't (this release). -->

| ID | User story | Priority |
|---|---|---|
| FR-001 | As a <persona>, I want <capability> so that <outcome>. | Must |

### Acceptance criteria

#### FR-001

- **Given** <context>, **when** <action>, **then** <observable result>.
- **Given** …

## 3. Non-functional requirements

<!-- Every NFR must be measurable and name how it will be verified in phase 8. -->

| ID | Category | Requirement | Verification method |
|---|---|---|---|
| NFR-001 | Performance | p95 latency of <operation> < <N> ms at <M> requests/s | Load test (k6) in ephemeral cluster |
| NFR-002 | Scalability | Supports <N> records of <entity> without degrading NFR-001 | Load test with seeded data |
| NFR-003 | Availability | Readiness reflects dependency health; no single request can crash the process | Integration + chaos test |
| NFR-004 | Observability | Structured JSON logs with correlation ID; RED metrics exposed | Automated test on log/metric output |
| NFR-005 | Portability | Runs on `linux/amd64` and `linux/arm64` | CI multi-arch build |
| NFR-006 | Maintainability | Unit test coverage of domain logic ≥ 80% | CI coverage gate |

## 4. Security requirements

<!-- Target: OWASP ASVS 5.0 Level 2. Record the ASVS requirement IDs each row satisfies. -->

| ID | Requirement | ASVS reference | Verification method |
|---|---|---|---|
| SR-001 | All endpoints except health checks require authentication | | Automated authz tests |
| SR-002 | Authorization is enforced server-side on every request, deny by default | | Automated authz tests |
| SR-003 | All input is validated against an allow-list schema at the API boundary | | Unit + DAST |
| SR-004 | Security-relevant events are logged without secrets or personal data | | Log review test |
| SR-005 | Secrets are never stored in the repository, image, or logs | | Secret scanning in CI |
| SR-006 | Dependencies with fixable HIGH/CRITICAL vulnerabilities block the build | | CI security gate |

## 5. Abuse cases

<!-- How would an attacker misuse the product? Each abuse case must map to at least one SR. -->

| ID | Attacker goal | Scenario | Mitigated by |
|---|---|---|---|
| AC-001 | | | SR- |

## 6. Data requirements

<!-- Inputs to phase 4. -->

| Entity | Expected volume (1 year) | Growth | Classification | Retention |
|---|---|---|---|---|
| | | | Public · Internal · Confidential · Restricted | |

## 7. Traceability

<!-- Filled progressively in phases 3, 4, 7, and 8. -->

| Requirement | Design element | Implementation | Test / evidence |
|---|---|---|---|
| FR-001 | | | |

## Gate checklist

- [ ] Every requirement has a stable ID and is testable
- [ ] Every Must-have story has acceptance criteria
- [ ] Every NFR is measurable and names its verification method
- [ ] Security requirements reference OWASP ASVS L2
- [ ] Every abuse case maps to a security requirement
- [ ] Data volumes and classification are estimated
