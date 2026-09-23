# 0003. Two versioned policy profiles: `standard` and `strict`

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-09-23 |
| **Deciders** | @Lewall-theart |

## Context

Verdicts are computed from facts and a policy ([ADR 0002](0002-staged-pipeline-facts-then-verdicts.md)). Different outputs need different bars: the PR gate must not block on debt nobody asked teams to fix yet, while an attestation sent to customers must make a strong, defensible claim ([charter §2](../00-charter.md#2-mission-and-the-decisions-it-serves)).

The name `baseline` was considered for the lower level, but it collides with the *reference snapshot* the gate compares against ([ADR 0004](0004-gate-rules-and-matrix.md)).

## Options considered

1. **One fixed policy** — simple, but either too strict for existing code or too weak for attestation.
2. **Fully free-form rules per organization** — maximum flexibility, but no shared meaning: "passes the policy" says nothing to a customer.
3. **A small set of named, versioned profiles**, with organization-level configuration only where it does not change what a profile *means*.

## Decision

We will ship two profiles, **`standard`** and **`strict`**, each identified by name and version (e.g. `strict@1`). Every requirement cites its source, with the *OpenSSF Compiler Options Hardening Guide for C and C++* as the primary reference.

| Property | `standard` | `strict` |
|---|---|---|
| PIE | Required for C/C++/Rust; warning for Go | Required for all languages |
| NX stack, no RWX segment, no TEXTREL | Required | Required |
| RELRO | Partial or better | Full |
| Stack canary | Required for C/C++ | Required for C/C++, strong variant |
| FORTIFY | Recommended | Required on glibc; `undeterminable` with reason on musl |
| CET SHSTK (x86_64) / BTI (aarch64) | Informational | Required as declared; effective reported separately |
| Dangerous RPATH/RUNPATH | Forbidden | Forbidden |
| setuid / file capabilities | Must be on an allowlist | Must be on an allowlist |

Evaluation rules common to both profiles:

- **Verdicts:** `pass`, `fail`, `waived` (valid exception), `not_applicable` (e.g. canary or FORTIFY for pure Go/Rust), `unknown` (fact is `undeterminable`).
- **`undeterminable` is never `fail` at a gate and never `pass` in an attestation.**
- The result records the profile name, version, and digest of the full policy used, so verdicts are reproducible.
- **Severity** is declared in the profile from three inputs: the property's class, exposure (image entrypoint > setuid/capability > regular binary > build-only tool), and language applicability. Severity **orders** findings in reports and backlogs; it **never decides** a gate (see [ADR 0004](0004-gate-rules-and-matrix.md)).

## Consequences

**Positive**

- "Meets `strict@1`" has one meaning for every customer and auditor.
- Profiles evolve by publishing a new version; old results stay interpretable.

**Negative**

- Organizations that want a middle level must wait for a new profile or use exceptions.
- Some `strict` requirements depend on toolchain support (e.g. CET/BTI marking); a toolchain that cannot produce them makes `strict` unreachable until upgraded.

**Follow-up actions**

- [ ] Map every profile requirement to its primary source in phase 1 requirements.
- [ ] Decide how to express "property not producible by this toolchain" (profile variant vs. organization-wide exception) — see [ADR 0004](0004-gate-rules-and-matrix.md).
