# 0007. `strict` adjusts for toolchain capability, under fail-safe rules

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-09-23 |
| **Deciders** | @Lewall-theart |

## Context

New binaries must meet `strict` ([ADR 0004](0004-gate-rules-and-matrix.md)). Some `strict` requirements can only be produced by a recent enough toolchain (e.g. CET/BTI marking). On an older toolchain, every new binary would fail the gate for a reason no developer can fix in their PR.

## Options considered

1. **Toolchain-aware `strict`** — requirements the detected toolchain cannot produce are evaluated as `not_applicable` with reason `toolchain_unsupported`.
2. **Organization-wide exception** — a special exception type declaring "our toolchain cannot produce X until date Y".

Option 2 is more explicit, but requires every organization to know and declare its toolchain limits up front. Option 1 works without configuration, at the risk of silently lowering the bar. That risk is addressed by the rules below.

## Decision

We will use option 1, with these rules:

1. **Only toolchain-dependent properties can be adjusted.** The profile lists them explicitly. Properties every supported toolchain can produce (PIE, NX, no RWX, no TEXTREL, RELRO, canary) are **never** adjusted.

   | Property | Capability depends on (to verify against primary sources in phase 1) |
   |---|---|
   | CET SHSTK/IBT marking | GCC ≥ 8, Clang ≥ 7 (`-fcf-protection`) |
   | BTI/PAC marking | GCC ≥ 9, Clang ≥ 8 (`-mbranch-protection`) |
   | FORTIFY level 3 | GCC ≥ 12 or recent Clang, and glibc ≥ 2.34 |
   | Stack clash protection | GCC ≥ 8, Clang ≥ 11 |
   | Automatic variable initialization | GCC ≥ 12, Clang ≥ 8 |

2. **Fail-safe on evidence.** The adjustment applies only when the toolchain is identified with certainty (e.g. from `.comment`). If the toolchain cannot be identified, the requirement stays: **no evidence, no adjustment**.
3. **All compilers must lack the capability.** A binary often records several compilers (main code plus vendored objects). The adjustment applies only if **every** recorded compiler lacks the capability. If the main toolchain supports it and one vendored object does not, the requirement stays and the finding points at the vendored object — exactly the regression Platform needs to see.
4. **Always disclosed.** The adjustment never disappears into a `pass`:

   ```yaml
   profile: strict@1
   toolchain_adjustments:
     - property: cet_shstk
       verdict: not_applicable
       reason: toolchain_unsupported
       detected_toolchain: ["GCC 7.5.0"]
   ```

   Reports show the count of adjusted binaries per property, and attestations list every adjustment next to waived items.
5. **Tracked as debt.** The share of binaries relying on an adjustment is a metric in the release report, so Platform sees the cost of not upgrading the toolchain.

## Consequences

**Positive**

- `strict` for new binaries is usable immediately, even on older toolchains, without configuration.
- A vendored object built with an old compiler is still caught, because of rule 3.
- Customers see which properties were not claimed and why.

**Negative**

- The engine must identify compilers and versions reliably; stripped `.comment` sections make the adjustment unavailable (by design, rule 2).
- The capability table must be maintained as toolchains evolve; it is versioned with the profile.
- Less explicit than an organization-declared exception: the decision is inferred, not declared.

**Follow-up actions**

- [ ] Verify every row of the capability table against compiler and glibc release notes in phase 1.
- [ ] Add corpus cases: old toolchain only; new toolchain with one old vendored object; stripped `.comment`.
