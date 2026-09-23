# 0006. Exceptions: in the service repository, reviewed by Security, always expiring

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-09-23 |
| **Deciders** | @Lewall-theart |

## Context

Some properties cannot be enabled for legitimate reasons, for example a JIT engine that requires memory that is both writable and executable. Developers need a legitimate path to proceed ([charter §3, P3](../00-charter.md#3-stakeholders-and-users)), and Security needs control and an audit trail. Customers receiving an attestation must see what was waived, not a false "pass".

## Options considered

1. **Central exception database managed by Security** — strong control, but a bottleneck, and exceptions drift away from the code they concern.
2. **Inline annotations in build files or source** — close to the code, but anyone who can edit the code can grant themselves an exception.
3. **A declarative file in the service repository, protected by CODEOWNERS** — close to the code, reviewed by Security, versioned in git history.

## Decision

We will use option 3: `.hardening/exceptions.yaml` in the service repository, with the file owned by the Security team in CODEOWNERS.

```yaml
schema: exceptions/v1
exceptions:
  - id: EXC-0007
    path: /opt/example/bin/jit-worker
    property: no_rwx_segment
    profiles: [standard, strict]
    justification: "JIT engine requires writable and executable pages"
    compensating_controls: ["seccomp profile", "runs as non-root"]
    ticket: https://tracker.example/SEC-123
    approved_by: "@security-reviewer"
    created: 2026-09-23
    expires: 2027-03-23
```

Rules:

- **`expires` is mandatory**, at most 12 months after `created`. The tool warns 30 days before expiry; after expiry the finding returns automatically.
- An exception is bound to **path + property**, not to a file hash, so it survives rebuilds.
- Path globs are allowed for `standard`; `strict` exceptions must name an exact path.
- A waived finding has verdict `waived`. Reports and attestations **list waived items explicitly**; they are never counted as `pass`.
- An exception that no longer matches any file is reported as **stale** so it can be removed.

## Consequences

**Positive**

- Developers have a documented path that does not require a meeting.
- Every exception has an approver, a reason, and an end date in git history.
- Customers see exactly what is waived.

**Negative**

- Security review becomes a dependency on PRs that add exceptions.
- Path binding means a file moved to a new path loses its exception (intended: the move deserves review).

**Follow-up actions**

- [ ] Write the JSON Schema for `exceptions/v1` and validate it in CI.
- [ ] Decide whether organization-wide exceptions (e.g. toolchain limitations, [ADR 0004](0004-gate-rules-and-matrix.md)) use this file format or a separate policy file.
