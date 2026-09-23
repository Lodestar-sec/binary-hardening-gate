# 06 · Engineering standards: <product-name>

## 1. Repository structure

<!-- Finalize once the stack is chosen in phase 2. Keep this layout unless an ADR says otherwise. -->

```
.
├── api/                 # OpenAPI contract (source of truth for the API)
├── src/                 # Application code — domain logic independent of frameworks
├── migrations/          # Versioned database migrations
├── tests/
│   ├── unit/
│   ├── integration/     # Against a real database
│   └── e2e/             # Against the deployed chart
├── deploy/
│   ├── helm/            # Helm chart — the only deployment artifact
│   └── policies/        # Admission policies used to verify releases
├── docs/                # Phase documents and ADRs
├── Dockerfile
└── compose.yaml         # Local development and `make demo`
```

## 2. Code standards

- Formatter and linter run in CI; warnings fail the build.
- No commented-out code, no TODOs without a linked issue.
- Public functions and API handlers have docstrings or equivalent.

## 3. Testing strategy

| Level | Scope | Runs | Target |
|---|---|---|---|
| Unit | Domain logic, no I/O | Every PR | ≥ 80% line coverage of domain code |
| Integration | Repository layer and migrations against a real database | Every PR | All queries in the query catalog covered |
| Contract | API responses validated against `api/openapi.yaml` | Every PR | 100% of endpoints |
| E2E | Critical user journeys against the ephemeral deployment | Every PR | Every Must-have FR |
| Security | SAST, SCA, secrets, IaC, image scan, DAST | Every PR | No fixable HIGH/CRITICAL |
| Performance | Load test against NFRs | Before each minor release | Every performance NFR |

## 4. Branching and reviews

- Trunk-based: short-lived branches from `main`, merged within 2 days.
- Squash merge only; the PR title (Conventional Commits) becomes the commit message.
- Every change goes through a pull request; `main` is protected by a ruleset.

## 5. Definition of Ready

A story can be started when:

- [ ] It has an ID, acceptance criteria, and priority
- [ ] Its design impact (API, data model, threat model) is understood

## 6. Definition of Done

A pull request can be merged when:

- [ ] Acceptance criteria are met and covered by tests
- [ ] All CI checks pass, including the ephemeral deployment and DAST
- [ ] Docs, API contract, and `CHANGELOG.md` are updated
- [ ] Threat model is updated if trust boundaries or data flows changed
- [ ] Traceability table in `01-requirements.md` is updated

## 7. Dependency policy

- Every new dependency is justified in the PR (why, alternatives, maintenance status, license).
- Lockfiles are committed; versions are pinned; Dependabot keeps them current.
- GitHub Actions are pinned to full commit SHAs.

## 8. CI/CD pipeline

| Trigger | Stage | Tooling | Blocking |
|---|---|---|---|
| PR, `main` | Filesystem scan (deps, secrets, IaC) | Trivy | Yes — fixable HIGH/CRITICAL |
| PR, `main` | Lint, unit, integration tests | <stack-specific, added in phase 6> | Yes |
| PR, `main` | Image build and scan | Buildx, Trivy | Yes — fixable HIGH/CRITICAL |
| PR, `main` | Ephemeral deployment | kind, Helm, `helm test` | Yes |
| PR, `main` | DAST baseline | OWASP ZAP | Yes — rules set to FAIL |
| `main` | Multi-arch image, SBOM, signature, provenance | Buildx, Syft, Cosign, GitHub attestations | Yes |
| Tag `vX.Y.Z` | Verify signature and provenance, deploy to a clean cluster with signature enforcement, promote tag, publish signed chart and release | Cosign, Kyverno, Helm, GHCR | Yes |

## 9. Environments

| Environment | Where | Lifetime | Purpose |
|---|---|---|---|
| Local | `docker compose` | On demand | Development and `make demo` |
| Ephemeral | kind on the CI runner | One pipeline run | Prove every change deploys and passes E2E/DAST |
| Release verification | kind on the CI runner, with Kyverno | One release | Prove the released artifacts are signed and deployable |

## Gate checklist

- [ ] Repository structure finalized for the chosen stack
- [ ] Stack-specific CI jobs added (lint, unit, integration, migrations, E2E)
- [ ] Walking skeleton passes the full pipeline end to end
