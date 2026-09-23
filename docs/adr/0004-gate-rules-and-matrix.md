# 0004. Gate rules: fixed invariants plus a configurable gate matrix

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-09-23 |
| **Deciders** | @Lewall-theart |

## Context

The charter fixes that the PR gate **blocks only regressions**, never pre-existing debt, and **only on certain evidence**, because false positives make developers disable the tool ([charter §2](../00-charter.md#2-mission-and-the-decisions-it-serves)).

Three cases are not covered by "regression only":

- A **brand-new internal binary** has no reference to regress from, so a weak new binary would pass.
- Files whose **owner cannot be determined** (`unknown`): blocking on them punishes developers for missing metadata; ignoring them hides risk. The right answer differs between organizations and between the PR gate and the release gate.
- **Distro or third-party** files that regress after a base-image change: the developer of the PR cannot fix them.

## Options considered

1. **Hard-coded rules** — predictable, but one organization's right answer for `unknown` or `distro_modified` is wrong for another.
2. **Everything configurable** — flexible, but a team could configure the gate to never block, and "blocking only on certain evidence" would stop being a guarantee.
3. **Fixed invariants + a configurable matrix** — the guarantees that make the gate trustworthy are fixed; the reaction per owner category and change kind is configurable, with safe defaults.

## Decision

We will use option 3.

### Fixed invariants (not configurable)

1. Only facts with **certain** confidence can produce `block`.
2. `undeterminable` never blocks.
3. A valid exception turns a finding into `waived`; it never blocks.
4. The gate compares against a **reference snapshot**: for the PR gate, the latest build of the main branch; for the release gate, the previous release.

### Change kinds

- **regression** — the verdict for a (path, property) got worse than in the reference snapshot.
- **new_fails_profile** — the binary did not exist in the reference snapshot and fails the profile set for new binaries.

**New binaries are evaluated against `strict`** by default ("clean as you code"): new code is held to the higher bar, existing code only must not get worse.

### Configurable gate matrix

Reaction per owner category ([ADR 0005](0005-ownership-detection.md)) × change kind: `block`, `warn`, or `ignore`. Defaults:

```yaml
gate:
  pr:
    new_binary_profile: strict
    matrix:
      internal:        { regression: block,  new_fails_profile: block }
      unknown:         { regression: warn,   new_fails_profile: warn  }
      third_party:     { regression: warn,   new_fails_profile: warn  }
      distro:          { regression: ignore, new_fails_profile: ignore }
      distro_modified: { regression: warn,   new_fails_profile: warn  }
  release:
    new_binary_profile: strict
    matrix:
      internal:        { regression: block,  new_fails_profile: block }
      unknown:         { regression: warn,   new_fails_profile: warn  }
      third_party:     { regression: warn,   new_fails_profile: warn  }
      distro:          { regression: warn,   new_fails_profile: warn  }
      distro_modified: { regression: block,  new_fails_profile: block }
```

The gate configuration is part of the policy: it is included in the policy digest recorded with every result, and changes to it are reviewed by Security through CODEOWNERS, so a team cannot silently switch its own gate to `ignore`.

### Output

The PR gate output (SARIF annotations, PR comment, exit code) is part of **v0.1**. SARIF is a thin view over `compare` (~4–6 hours) and makes the gate usable in GitHub code scanning immediately.

## Consequences

**Positive**

- The guarantee "the gate only blocks on certain evidence" holds for every organization.
- Organizations can start permissive (`warn`) on `unknown` and tighten later without a new tool version.
- Distro regressions after a base-image change reach Platform through the release report instead of blocking unrelated PRs.

**Negative**

- New binaries against `strict` will fail where the toolchain cannot produce a `strict` property (e.g. CET/BTI marking on an old compiler). Until that is handled, teams will need exceptions for it.
- The matrix adds configuration that must be documented and tested.

**Follow-up actions**

- [ ] Decide how to handle properties the toolchain cannot produce: a `strict` variant keyed on detected toolchain capability, or an organization-wide exception type.
- [ ] Define "latest build of main" precisely (artifact store, how the reference snapshot is fetched in CI).
- [ ] Test every default in the matrix with synthetic facts.
