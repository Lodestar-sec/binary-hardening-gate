# 0005. Ownership detection for every ELF file

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-09-23 |
| **Deciders** | @Lewall-theart |

## Context

Every finding must have an owner and an action ([charter §2](../00-charter.md#2-mission-and-the-decisions-it-serves)). In a typical image most ELF files come from distro packages, which the product team cannot fix directly; only a minority are built internally. The gate matrix ([ADR 0004](0004-gate-rules-and-matrix.md)) and the release report both depend on knowing which is which.

## Options considered

1. **Path conventions only** (e.g. everything under `/opt` is internal) — cheap, but wrong often enough to mislead the gate.
2. **Explicit ownership map only** — accurate for what is listed, but everything else stays unknown, including the thousands of distro files.
3. **Layered signals with an explicit order**, where a declared ownership map wins and package metadata fills in the rest.

## Decision

We will classify every file into `internal`, `distro`, `distro_modified`, `third_party`, or `unknown`, using these signals in order; the first match wins:

| # | Signal | Result |
|---|---|---|
| 1 | Ownership map in the repository (path globs → team, like CODEOWNERS) | `internal` + team |
| 2 | Distro package database: dpkg (`/var/lib/dpkg/info/*.list`, or `/var/lib/dpkg/status.d/` in distroless images), rpmdb, apk (`/lib/apk/db/installed`) | `distro` + package name and version |
| 3 | Language package metadata: Python wheels (`*.dist-info/RECORD`), `node_modules/*/package.json` | `third_party` + package name |
| 4 | Layer history in the image config (`COPY` vs. `RUN`) | hint only, low confidence |
| 5 | No match | `unknown` |

Example ownership map (synthetic values):

```yaml
schema: ownership/v1
owners:
  - paths: ["/opt/example/bin/**", "/opt/example/lib/**"]
    team: "@example-org/edr-agent"
  - paths: ["/usr/local/bin/example-cli"]
    team: "@example-org/platform"
```

**Integrity check:** when the package database provides per-file digests (dpkg `md5sums`, rpm file digests), the file's hash is compared. A mismatch classifies the file as **`distro_modified`**: something replaced a package-owned file after installation, which is worth investigating.

`unknown` files are handled by the gate matrix (default `warn`), and the share of `unknown` files is reported as a metric so teams can reduce it by extending the ownership map.

## Consequences

**Positive**

- Findings route to the party who can act: service team, Platform, or upstream.
- Distro files stop drowning internal findings in reports.
- `distro_modified` surfaces tampered or hand-patched package files as a by-product.

**Negative**

- Several package database formats must be parsed (dpkg, rpm, apk, distroless variants), each with its own edge cases.
- The package database inside an image can itself be missing or wrong; ownership is therefore a claim with a confidence level, not a certainty.

**Follow-up actions**

- [ ] Measure in Phase 0 research what share of ELF files each signal classifies on public images.
- [ ] Decide which package database formats ship in v0.1 (proposed: dpkg and apk first).
