# 0002. One staged pipeline: facts first, verdicts second

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-09-23 |
| **Deciders** | @Lewall-theart |

## Context

The product must answer different decisions (PR gate, release gate, backlog, attestation) from the same analysis, each with its own confidence threshold ([charter §2](../00-charter.md#2-mission-and-the-decisions-it-serves)). It must also:

- re-use analysis results across images, because the same library appears in many images (images with 10,000+ ELF files);
- explain in a diff whether a verdict changed because **the binary changed** or because **the policy changed**;
- be reproducible: the same inputs and the same policy version must give the same verdicts;
- let the analysis engine be tested against a ground-truth corpus independently of policy.

Tools like `checksec` compute a property and judge it in the same step, so none of the above is possible.

## Options considered

1. **Single step** — each check reads the binary and emits pass/fail directly. Simple, but facts and judgement are entangled: a policy change requires a full rescan, results cannot be cached across policies, and diffs cannot attribute the cause.
2. **Two separate tools** — an analyzer and a policy evaluator shipped and run independently. Clean boundary, but two things to install, version, and wire together in CI; too much overhead for users and for a 2 h/week maintainer.
3. **One pipeline, explicit stages with data contracts between them** — one command runs everything, but each stage only consumes the previous stage's output.

## Decision

We will build **one pipeline with four stages**. `scan` runs all of them by default; the boundaries are versioned data contracts, not separate products.

```
Stage 0  Ingest     image / rootfs / file  →  file inventory + context
                    (paths, mode bits, xattrs, image config, owner)
Stage 1  Analyze    file bytes             →  FACTS
                    (present / absent / undeterminable, evidence, confidence, rule version)
Stage 2  Evaluate   facts + policy         →  VERDICTS
                    (pass / fail / waived / not_applicable / unknown)
Stage 3  Render     verdicts (+ diff)      →  views: JSON, Markdown, SARIF, CSV, attestation
```

Two invariants make the boundaries real:

- **Stage 1 never reads policy.** Facts depend only on file bytes and rule versions.
- **Stage 2 never reads file bytes.** Verdicts are a pure function of facts, context, and policy, so they can be recomputed in milliseconds.

## Consequences

**Positive**

- Facts are cached by file content hash and re-used across images and across policy changes.
- Adding or expiring an exception, or changing a profile, only re-runs stages 2–3.
- A diff can report *bytes changed*, *rule version changed*, or *policy changed* as the cause.
- The engine is tested against the corpus with no policy involved; policy is tested with synthetic facts and no binaries.

**Negative**

- The facts schema is a public contract early on; changing it later needs a schema version and migration.
- Slightly more design work up front than a single-step checker.

**Follow-up actions**

- [ ] Define the facts schema (versioned JSON) in phase 4, including evidence records.
- [ ] Decide in the delivery plan whether `evaluate` (stages 2–3 from stored facts) ships in v0.1 or v0.2.
