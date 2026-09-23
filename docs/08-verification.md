# 08 · Verification: <product-name>

Evidence that the product meets its requirements. Every row links to a CI run, report, or test.

## 1. Functional verification

| Requirement | Test(s) | Result | Evidence |
|---|---|---|---|
| FR-001 | | ✅ / ❌ | <CI run link> |

## 2. Non-functional verification

| Requirement | Method | Target | Measured | Result | Evidence |
|---|---|---|---|---|---|
| NFR-001 | Load test (k6) | | | | |

## 3. Security verification

| Check | Tool / method | Result | Evidence |
|---|---|---|---|
| Static analysis (SAST) | | | |
| Dependency and image vulnerabilities | Trivy | | |
| Secrets | Trivy | | |
| IaC / Kubernetes misconfiguration | Trivy | | |
| Dynamic analysis (DAST) | OWASP ZAP baseline | | |
| OWASP ASVS L2 checklist | Manual review | <n>/<total> passed | |
| Supply chain | OpenSSF Scorecard | <score> | |
| Signature and provenance | Cosign, `gh attestation verify` | | |

### Security requirements

| Requirement | Verified by | Result |
|---|---|---|
| SR-001 | | |

## 4. Open findings

| Finding | Severity | Decision (fix / accept) | Owner | Due |
|---|---|---|---|---|
| | | | | |

## Gate checklist

- [ ] Every Must-have FR verified
- [ ] Every NFR measured against its target
- [ ] Every SR verified; no open HIGH/CRITICAL findings without an accepted risk
