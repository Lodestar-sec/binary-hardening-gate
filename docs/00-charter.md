# 00 · Project charter: Binary Hardening Gate *(working title)*

| | |
|---|---|
| **Status** | Draft |
| **Owner** | @Lewall-theart |
| **Last updated** | 2026-09-23 |

## 1. Problem statement

Vendors that ship self-hosted software (EDR agents, network appliances, databases distributed as containers) are asked by customers, pentesters, and regulators (EU CRA) whether their binaries are built with exploit mitigations such as PIE, RELRO, stack canaries, and CET/BTI.

Today they cannot answer with confidence:

- **Coverage is partial.** Teams run `checksec` by hand on a few main binaries before a major release and paste the output into a spreadsheet. Shared libraries and third-party binaries are skipped, although a single image can contain ~1,400 ELF files (10,000+ in CUDA images).
- **Regressions are silent.** A base-image upgrade, a compiler change, a switch to musl or static linking, or a vendored Makefile that overrides `CFLAGS` can drop a mitigation. Nobody notices until a pentest months later.
- **Answers are not verifiable.** A spreadsheet is a claim, not evidence. Customers and auditors cannot check it independently.
- **Existing tools report per file and declared flags only.** They do not model what the running process actually gets once libraries, the loader, libc, and the kernel are taken into account, and they have no "cannot determine" state, so stripped static binaries produce false results.

> **Evidence still to collect (Phase 0 research):** measure the problem in the wild on a sample of public images: number of ELF files per image, share owned by distro packages vs. vendor code, and how often declared hardening differs between consecutive releases. Findings replace the estimates above.

## 2. Mission and the decisions it serves

**Mission:** help organizations that ship software **know** the hardening state of every binary they deliver, **prevent regressions** before release, and **prove** it to customers with independently verifiable evidence.

Every feature must serve at least one of the three verbs: *know*, *prevent regressions*, *prove*. A tool is only valuable when someone makes a decision with its output:

| Decision | Who decides | When | What they need from the tool |
|---|---|---|---|
| Can this PR be merged? | Developer + CI gate | Every PR | Which binaries regressed against the main branch, and the flag that fixes it |
| Can this release ship? | Release manager / Product Security | Before each release | Diff against the previous release, open findings, valid exceptions |
| Should we adopt this base image / toolchain? | Platform engineer | Debian/GCC upgrade, musl or static switch | Old vs. new comparison, and where each change comes from |
| Should this exception be granted? | Security | When a team requests one | Justification, scope, expiry, approver |
| How do we answer this customer? | Product Security | Security questionnaire, audit | Report and signed attestation |
| What do we fix first? | Security + teams | Backlog planning | Findings ranked by severity **and grouped by who can fix them** |

### Principles derived from these decisions

1. **The diff is the core product; a scan is raw material.** Four of the six decisions compare two snapshots (PR vs. main, release vs. previous release, old vs. new base image).
2. **Each output has its own confidence threshold**, because each decision pays a different price for a wrong answer:
   - *PR gate:* a false positive is very expensive (developers will disable the tool). The gate **blocks only regressions against the main branch**, never pre-existing debt, and **only on certain evidence**.
   - *Attestation:* a false negative is very expensive (it is a signed promise). It **claims "present" only with high confidence**; anything undeterminable is listed as such, never rounded up.
   - *Backlog:* medium confidence is acceptable if the ranking is right.
3. **Every finding has an owner and an action.** A finding nobody can act on is noise. Actions:
   - **Fix build**: add a flag. Owner: the service team.
   - **Upgrade base**: base image or toolchain. Owner: Platform.
   - **Report upstream / to vendor**: distro or third-party binary. Owner: Product Security.
   - **Request exception**: justification, scope, and **mandatory expiry date**. Owner: team, approved by Security.
   - **Accept / informational**: no action.
4. **Every conclusion is backed by verifiable evidence.** See [§5](#5-evidence-model).

## 3. Stakeholders and users

| Persona | Role | What they need from this product | How they do it today |
|---|---|---|---|
| **P1** Product Security Engineer | Answers customer questionnaires, pentest reports, CRA obligations | An answer for **every** ELF in a release, with signed evidence to send to customers | `checksec` by hand on a few binaries, pasted into a spreadsheet |
| **P2** Platform / Build Engineer | Owns base images and toolchains for ~150 C/C++/Go/Rust services | Compare hardening between old and new base image before rollout, and know which toolchain change caused a regression | No way to know; found by pentest months later |
| **P3** C/C++ Developer | Has a PR blocked by the gate | Which binary, which property, why it matters in two lines, the exact flag for CMake / Make / Cargo / Go, and a legitimate exception path when the property cannot be enabled (e.g. a JIT that needs RWX memory). Fixed in 15 minutes without asking Security | — |
| **P4** Customer / Auditor (read-only) | Verifies vendor claims | A signed attestation attached to the image stating which profile every binary meets, verifiable with `cosign verify-attestation` | Trusts a spreadsheet |

**Anti-persona:** malware analysts and reverse engineers who need a disassembler or decompiler. They already have Ghidra and IDA.

## 4. Output contract (per persona)

There is **one core result** (versioned JSON). Every output below is a **view** of it, differing only in scope and confidence threshold.

| Output | Persona | Scope | Confidence threshold | Format |
|---|---|---|---|---|
| PR gate | P3 | Regressions vs. main only | Certain | SARIF, PR comment, exit code |
| Image / toolchain diff | P2 | All changes | Medium and above, labelled | Markdown/HTML, JSON |
| Release report | P1 | Entire release | All levels, labelled | HTML/Markdown, CSV |
| Attestation | P4 | Entire release | High | in-toto, signed with Cosign |

### P3: Developer, when a PR is gated

Questions: Did my PR make any binary weaker than main? Which binary, which property, before → after? Why does it matter? How do I fix it in my build system? Where do I request an exception?

Each entry contains: binary path, property, before → after, the evidence line with a reproduce command, a fix snippet per build system, and a link to the exception process.

### P2: Platform engineer, when changing base image or toolchain

Questions: How did hardening change between image A and B? Where did each change come from: package version change, toolchain change, new file, removed file? Which part is ours to fix, which belongs to the distro or a third party?

Report grouped by **owner** first, then by **cause**, with a summary of files improved, regressed, added, removed.

### P1: Product Security, before a release and when answering a questionnaire

Questions: What is the state of every ELF in release X? Which findings are open, owned by whom, with which action? Which exceptions are valid or expired? How do we answer "does the product use PIE/RELRO/canary/CET?"

Report contains: coverage table (% of ELF files meeting each property, split by owner category: internal, distro, third party), findings grouped by action and owner, exceptions with expiry, and a questionnaire export where every answer has a number, a scope, and a pointer to evidence.

### P4: Customer / Auditor

Questions: Is this attestation genuine and bound to the digest of the image I received? Which profile, profile version, and kernel assumptions does it claim? Can I verify each claim myself?

Attestation contains: profile ID and version, tool version, kernel assumptions, summary, and the evidence records (or the digest of an evidence bundle). Undeterminable items are listed separately with their reason.

## 5. Evidence model

"Proof" in this product means **verifiable evidence**, not exploit code. The tool never generates exploits and never executes the binaries it analyzes. Auditors need to check conclusions, not see a binary attacked.

Every conclusion (`present` / `absent` / `undeterminable`) carries an evidence record:

1. **File identity**: path inside the image, SHA-256, build-id, layer that contains it.
2. **Location**: section or segment, file offset, virtual address.
3. **Raw data**: the bytes that were read and their decoded value (e.g. `DT_FLAGS_1 = 0x08000001` → `NOW | PIE`).
4. **Interpretation rule**: rule ID and rule version that turned raw data into a conclusion, and the resulting confidence.
5. **Reproduce command**: a standard, tool-independent command (`readelf -d`, `readelf -l`, `readelf -n` …) so anyone can check the conclusion without trusting this tool.

Consequences:

- `undeterminable` also has evidence: the reason it could not be decided (e.g. "static, stripped, no annobin or DWARF").
- The attestation signs both the conclusions and the evidence records: Cosign proves the signature, binutils proves the content.
- Evidence is a first-class entity in the data model, so a diff can tell whether a conclusion changed because **the bytes changed** or because **the rule version changed**.

## 6. Objectives and success metrics

| Objective | Metric | Baseline (today) | Target | How it is measured |
|---|---|---|---|---|
| Know | Share of ELF files in a release with a result (including `undeterminable` with a reason) | Unknown; manual spot checks | 100% | Result count vs. ELF count found by magic bytes |
| Prove | Time to answer a hardening section of a security questionnaire | Days (estimate, to confirm in Phase 0) | < 1 hour | Timed walkthrough on a sample questionnaire |
| Prove | Conclusions reproducible with binutils | 0% | 100% of `present`/`absent` conclusions | Automated check in CI replays every reproduce command on the test corpus |
| Prevent regressions | Gate false-positive rate | n/a | < 2% *(to confirm)* | Regressions flagged vs. confirmed on the test corpus and real PRs |
| Prevent regressions | Regressions caught before release vs. found after | Unknown | Tracked from v0.2 | Snapshot history in the database |

## 7. Scope

### In scope

- **Inputs:** OCI image (registry reference, tarball, OCI layout; each platform of a multi-platform image), rootfs directory, single file, release tarball.
- **File detection** by ELF magic bytes, not by extension; correct classification of executables, PIE, shared libraries, static and static-PIE; `ET_REL` and `ET_CORE` skipped.
- **Architectures (v1):** x86_64 and aarch64, 64-bit little-endian. Parser is architecture-independent; checks are architecture-aware.
- **Language and libc awareness:** glibc vs. musl, Go, Rust, GCC/Clang, so that policy does not misreport memory-safe languages or musl images.
- **Image semantics:** final filesystem, file mode bits and xattrs (setuid, setgid, file capabilities), image config (entrypoint, user, environment).
- **Library resolution inside the image**, chroot-style, never following links outside the image root.
- **Declared vs. effective** protection for each executable, given its loaded libraries, libc, loader, and a stated kernel profile.
- **Diff** between two results, and the outputs in [§4](#4-output-contract-per-persona).

### Out of scope (non-goals)

- Finding vulnerabilities in source code (SAST, fuzzing tools do this).
- CVE scanning of packages (Trivy, Grype do this).
- Reverse engineering, disassembly UI, decompilation (Ghidra, IDA do this).
- Generating exploits or proof-of-concept attacks, or executing analyzed binaries.
- Assessing the kernel or container runtime. The kernel is an **input assumption**, stated in every report.
- `.deb` / `.rpm` packages as input, and architectures other than x86_64/aarch64, before v1.

## 8. Assumptions and constraints

| Type | Statement |
|---|---|
| Constraint | Budget: $0 — open-source components and free tiers only |
| Constraint | Maintainer time: ~2 hours per week (~8–9 hours per month) |
| Constraint | Must run on `linux/amd64` and `linux/arm64` |
| Constraint | The engine must withstand malformed or hostile ELF files: per-file time and memory limits, never execute or load analyzed binaries, parser is fuzzed |
| Constraint | The core engine (ELF parsing, check logic) is written by the owner, not generated, so every line can be explained |
| Assumption | Kernel version and configuration of the production environment are provided as a profile, not detected |
| Assumption | Most deployed containers are x86_64 or aarch64 |

## 9. Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Scope grows faster than 2 h/week can deliver; project stalls | High | High | Strict v0.1 / v0.2 split; every feature must map to a decision in §2; re-estimate at each phase gate |
| False positives in the gate make developers disable the tool | Medium | High | Gate blocks only regressions, only on certain evidence; FP rate is a tracked metric |
| Attestation claims something that is not true | Low | High | High-confidence threshold; `undeterminable` never rounded up; evidence reproducible with binutils |
| The parser itself is exploited by a crafted ELF | Medium | High | Memory-safe implementation language, fuzzing, resource limits, no execution; covered in threat model (phase 5) |
| Test corpus contains third-party binaries that cannot be redistributed | Medium | Medium | Build the corpus from source with Dockerfiles across compiler × libc × flag matrix; this also gives ground truth |
| Loader / libc / kernel behavior differs across versions and is modeled wrong | Medium | Medium | Rules are versioned; each rule cites its primary source (ABI spec, glibc/kernel source) |

## 10. High-level milestones

Estimates assume ~2 hours per week and will be revised at each gate in [07-delivery-plan.md](07-delivery-plan.md).

| Milestone | Outcome | Target date |
|---|---|---|
| Phase 0 | Charter approved; evidence research on public images done | 2026-10 |
| Phase gates 1–5 | Requirements, technology selection, architecture, data model, threat model approved | 2027-01 |
| Walking skeleton | First commit through the full pipeline | 2027-02 |
| v0.1 (CLI) | `scan`, `compare`, `explain`; structural ELF checks; core JSON schema with evidence, owner, confidence, rule version; PR gate with SARIF output; test corpus | 2027-06 |
| v0.2 (service + DB) | Snapshot history, regression diff across releases, full OCI layer handling, effective protection | 2027-11 |
| Later | Disassembly heuristics, annobin, attestation | — |

## 11. Decisions and open questions

### Settled (policy layer)

| Topic | Decision | ADR |
|---|---|---|
| Pipeline | One pipeline, four stages: ingest → analyze (facts) → evaluate (verdicts) → render | [0002](adr/0002-staged-pipeline-facts-then-verdicts.md) |
| Profiles | `standard` and `strict`, versioned, sourced from the OpenSSF hardening guide | [0003](adr/0003-policy-profiles.md) |
| Gate | Fixed invariants + configurable matrix (owner × change kind); new binaries must meet `strict`; SARIF in v0.1 | [0004](adr/0004-gate-rules-and-matrix.md) |
| Ownership | `internal` / `distro` / `distro_modified` / `third_party` / `unknown` from ordered signals | [0005](adr/0005-ownership-detection.md) |
| Exceptions | `.hardening/exceptions.yaml` in the service repo, CODEOWNERS Security, max 12 months, bound to path + property | [0006](adr/0006-exceptions.md) |
| Toolchain limits | `strict` adjusts toolchain-dependent properties only, only on certain toolchain evidence, only if every recorded compiler lacks the capability; always disclosed | [0007](adr/0007-strict-toolchain-adjustment.md) |

### Open

1. **Implementation language:** decided in phase 2 with an ADR, per the org lifecycle.

## Gate checklist

- [ ] The problem is described with evidence, not assumptions *(Phase 0 research pending)*
- [ ] Every objective has a baseline, a target, and a measurement method *(baselines pending Phase 0)*
- [x] Non-goals are listed
- [x] Top risks have a mitigation
