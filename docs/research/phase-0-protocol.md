# Phase 0 measurement protocol

| | |
|---|---|
| **Status** | Pre-registered 2026-09-23 — frozen together with [phase-0-plan.md](phase-0-plan.md) at tag `phase-0-prereg` (appendix A completed in T1, before any comparison runs) |
| **Owner** | @Lewall-theart |
| **Last updated** | 2026-09-23 |

This protocol gives every term in the hypotheses ([plan §4](phase-0-plan.md#4-hypotheses-predictions-and-decision-rules)) an **operational definition**: which bytes are read, which values are possible, how they are counted, and what the denominator is. A hypothesis verdict is only valid if it was computed exactly as defined here. Any departure is a deviation ([plan §5.3](phase-0-plan.md#53-pre-registration)).

Constants follow the ELF gABI, the x86-64 and AArch64 psABIs, and the GNU extensions as implemented in binutils. Where this protocol states a runtime meaning that is not yet verified against a primary source, it is marked **[verify M25]** and must be confirmed during desk research before the analysis step.

## 1. Units of analysis

| Unit | Definition | Used for |
|---|---|---|
| **Image instance** | (registry reference, manifest digest, platform) | Everything |
| **File instance** | (image instance, canonical path) — one per regular file in the final filesystem | Counts, prevalence |
| **Content unit** | SHA-256 of the file bytes | Deduplication (M02), caching |
| **Observation** | (file instance, property) → one fact | Prevalence, regressions |
| **Release pair** | (product, release *n*, release *n+1*) | H1.3, H1.4, 2A, H3.2–H3.4 |

**Canonical path.** The path after resolving every symbolic link *inside the image root* (never outside it). Example: on merged-`/usr` images, `/bin/ls` and `/usr/bin/ls` are one file instance with canonical path `/usr/bin/ls`. Hard links within one image are counted once; aliases are recorded.

**Final filesystem.** The result of applying all layers in order, including whiteouts (`.wh.<name>`) and opaque directories (`.wh..wh..opq`). Files deleted by a later layer are not in scope.

## 2. What counts as an in-scope ELF file

A file instance is **in scope** if all of the following hold. Otherwise it is recorded in the inventory with the reason, and excluded from every denominator except M01's inventory counts.

| Check | Field | In-scope value | Otherwise |
|---|---|---|---|
| Regular file | tar entry type | `0` (regular) or hard link to one | not an ELF candidate |
| Magic | bytes 0–3 | `7F 45 4C 46` | not ELF |
| Class | `e_ident[EI_CLASS]` | `2` (ELFCLASS64) | `out_of_scope_class` |
| Byte order | `e_ident[EI_DATA]` | `1` (little-endian) | `out_of_scope_endian` |
| Machine | `e_machine` | `62` (EM_X86_64) or `183` (EM_AARCH64), matching the image platform | `unsupported_arch` |
| Type | `e_type` | `2` (ET_EXEC) or `3` (ET_DYN) | `1` ET_REL / `4` ET_CORE → `out_of_scope_type` |
| Parseable | headers within file bounds | yes | `parse_error` (exclusion log) |

Extension and location are never used to decide whether a file is ELF.

## 3. Classification (`elf_kind`)

Inputs: `e_type`, presence of `PT_INTERP` (p_type `3`), `DT_FLAGS_1` (tag `0x6ffffffb`) bit `DF_1_PIE` (`0x08000000`), presence of `DT_SONAME` (tag `14`), presence of `PT_DYNAMIC` (p_type `2`).

Rules are applied in order; the first match wins.

| # | Condition | `elf_kind` | Confidence |
|---|---|---|---|
| 1 | `ET_EXEC`, no `PT_INTERP`, no `PT_DYNAMIC` | `static` | certain |
| 2 | `ET_EXEC`, `PT_INTERP` present | `exec` (dynamic, non-PIE) | certain |
| 3 | `ET_DYN`, `DF_1_PIE` set, `PT_INTERP` present | `pie` | certain |
| 4 | `ET_DYN`, `DF_1_PIE` set, no `PT_INTERP` | `static_pie` | certain |
| 5 | `ET_DYN`, no `DF_1_PIE`, `DT_SONAME` present | `shared_lib` (runnable if `PT_INTERP` present, e.g. `libc.so.6`) | certain |
| 6 | `ET_DYN`, no `DF_1_PIE`, no `DT_SONAME`, `PT_INTERP` present | `pie` (older linkers do not set `DF_1_PIE`) | high |
| 7 | `ET_DYN`, no `DF_1_PIE`, no `DT_SONAME`, no `PT_INTERP` | `shared_lib` | high |
| 8 | Anything else | `unclassified` | none — excluded with reason |

The dynamic loader itself (`ET_DYN`, no `PT_INTERP`, `DT_SONAME` matching `ld-linux*` or `ld-musl*`) is classified `shared_lib` with flag `is_loader = true`.

**Executable kinds** = `exec`, `pie`, `static`, `static_pie`. **Dynamic objects** = files with `PT_DYNAMIC`.

## 4. Confidence levels

| Level | Definition |
|---|---|
| **certain** | Decided only by ELF structures whose meaning is fixed by the gABI, psABI, or GNU extension specifications: ELF header, program headers, dynamic tags, GNU property notes, build-id note |
| **high** | Decided by the dynamic symbol table (imports and exports by exact name), or by recorded build metadata (annobin notes, `DW_AT_producer` with recorded switches, Go build info) |
| **medium** | Decided by a heuristic: ratios, name patterns, or indirect indicators |
| **none** | Not decidable from the file → fact state `undeterminable` with a reason code |

Fact states are `present`, `absent`, `undeterminable`, or `not_applicable`. Unless a hypothesis states otherwise, **only `certain` facts count toward thresholds**; `high` and `medium` facts are reported separately.

## 5. Property definitions

For each property: where it applies, how it is extracted, the values it can take, and its confidence.

### 5.1 Memory layout and segments

| Property | Applies to | Extraction | Values | Confidence |
|---|---|---|---|---|
| `pie` | Executable kinds | From `elf_kind`: `pie`, `static_pie` → present; `exec`, `static` → absent | present / absent | as the classification rule (§3) |
| `nx_stack` | All in-scope files | `PT_GNU_STACK` (`0x6474e551`): `p_flags & PF_X (0x1)` = 0 → `nx`; ≠ 0 → `exec`; segment missing → `missing` | nx / exec / missing | certain |
| `rwx_segment` | All in-scope files | Any `PT_LOAD` (p_type `1`) with both `PF_W (0x2)` and `PF_X (0x1)` | none / present (+ count) | certain |
| `textrel` | Dynamic objects | `DT_TEXTREL` (tag `22`) present, or `DT_FLAGS` (tag `30`) & `DF_TEXTREL (0x4)` | absent / present | certain |
| `relro` | Dynamic objects and `static_pie` | `PT_GNU_RELRO` (`0x6474e552`) absent → `none`; present → `partial`; present **and** (`DT_BIND_NOW` (tag `24`) present, or `DT_FLAGS & DF_BIND_NOW (0x8)`, or `DT_FLAGS_1 & DF_1_NOW (0x1)`) → `full` | none / partial / full | certain |

The runtime meaning of `nx_stack = missing` differs by architecture and kernel version **[verify M25]**. Phase 0 records `missing` as its own value and treats it as failing `standard`; the verified meaning is reported next to the counts.

### 5.2 Control-flow marking (GNU property notes)

Read `PT_GNU_PROPERTY` (`0x6474e553`) if present, otherwise every `PT_NOTE` (p_type `4`) with owner `"GNU"` and type `NT_GNU_PROPERTY_TYPE_0` (`5`).

| Property | Applies to | Extraction | Values | Confidence |
|---|---|---|---|---|
| `cet_ibt`, `cet_shstk` | x86_64 in-scope files | `GNU_PROPERTY_X86_FEATURE_1_AND` (`0xc0000002`): bit `0x1` = IBT, bit `0x2` = SHSTK. No property note, or property absent → absent | present / absent | certain |
| `bti`, `pac` | aarch64 in-scope files | `GNU_PROPERTY_AARCH64_FEATURE_1_AND` (`0xc0000000`): bit `0x1` = BTI, bit `0x2` = PAC | present / absent | certain |

These are **declared** markings. Effective protection (H3.6) is defined in §10.

### 5.3 Library search paths

| Property | Applies to | Extraction | Values | Confidence |
|---|---|---|---|---|
| `rpath`, `runpath` | Dynamic objects | Strings of `DT_RPATH` (tag `15`) and `DT_RUNPATH` (tag `29`), split on `:` | list of entries | certain |
| `search_path_risk` | Dynamic objects with any entry | Each entry classified, first match wins: `empty` (zero-length entry) → risky; `relative` (does not start with `/` or `$ORIGIN`) → risky; contains `$ORIGIN` → `origin` (recorded, not risky by itself); absolute and the directory does not exist in the final filesystem → risky (`missing`); absolute and writable by the image's configured user (§7) → risky (`writable`); otherwise `ok` | none / risky (+ entries and reasons) | certain |

### 5.4 Stack protection and FORTIFY (not counted toward "certain" thresholds)

| Property | Applies to | Extraction | Values | Confidence |
|---|---|---|---|---|
| `canary` | Dynamic objects whose language includes C or C++ | Undefined symbol in `.dynsym` named `__stack_chk_fail` or `__stack_chk_fail_local`, or on aarch64 `__stack_chk_guard` → present. None of them → absent | present / absent | high |
| `canary` | `static`, `static_pie` | Not decidable in Phase 0 (no disassembly) unless annobin notes or `DW_AT_producer` record the switch | from metadata, else undeterminable (`static_no_metadata`) | high / none |
| `fortify` | Dynamic objects linked against glibc | Count undefined `.dynsym` symbols in the fortified set (names of the form `__<fn>_chk` listed in the glibc version maps) and imports of their unfortified counterparts. ≥ 1 `_chk` import → present; 0 `_chk` imports and ≥ 1 fortifiable import → absent; no fortifiable import → not_applicable | present / absent / not_applicable (+ both counts) | medium |
| `fortify` | musl-linked files | Not decidable from symbols (musl uses header-only checks) | undeterminable (`musl_fortify`) | none |
| `fortify_level` | Any | Only from annobin notes or `DW_AT_producer` switches | 1 / 2 / 3 / undeterminable | high / none |

Canary presence says nothing about coverage (how many functions are protected); coverage is out of scope for Phase 0.

### 5.5 Build provenance

| Field | Extraction |
|---|---|
| `stripped` | No section named `.symtab` |
| `build_id` | `PT_NOTE`, owner `"GNU"`, type `NT_GNU_BUILD_ID` (`3`), hex of descriptor |
| `comment_compilers` | Section `.comment`, split on NUL; each string matched against: `GCC: \(.*\) (\d+\.\d+(\.\d+)?)`, `clang version (\d+\.\d+(\.\d+)?)`, `rustc version (\d+\.\d+\.\d+)`; unmatched strings kept verbatim |
| `go_version` | Section `.go.buildinfo` (or a segment containing the 14-byte magic `FF 20 47 6F 20 62 75 69 6C 64 69 6E 66 3A`), version string decoded per the Go build-info format |
| `annobin` | Presence of section `.gnu.build.attributes` or `.annobin.notes` |
| `dwarf_producer` | `DW_AT_producer` of the first compilation unit in `.debug_info`, if present |
| `libc` | `DT_NEEDED` contains `libc.so.6` → `glibc`; `PT_INTERP` or `DT_NEEDED` matches `ld-musl-*` / `libc.musl-*` → `musl`; static → `unknown` unless `comment_compilers` or symbols identify it |
| `language` | Rules in order, several may apply (e.g. cgo): `go_version` present → `go`; any compiler string or symbol identifies `rustc` → `rust`; `DT_NEEDED` contains `libstdc++.so.*` or `libc++.so.*` → `cpp`; linked against glibc or musl and none of the above → `c`; otherwise `unknown` |

### 5.6 File system attributes

From the tar header of the layer entry that produced the file in the final filesystem.

| Field | Extraction |
|---|---|
| `setuid`, `setgid` | Mode bits `04000`, `02000` |
| `capabilities` | PAX header `SCHILY.xattr.security.capability` present → decoded capability set |
| `uid`, `gid`, `mode` | Tar header fields |

## 6. Ownership

Signals as in [ADR 0005](../adr/0005-ownership-detection.md). In public images no ownership map exists, so Phase 0 uses signals 2, 3, and 4 only.

| Source | How files are listed | Integrity data |
|---|---|---|
| dpkg | `/var/lib/dpkg/info/<pkg>[:<arch>].list`; distroless: `/var/lib/dpkg/status.d/<pkg>` | `<pkg>.md5sums`: MD5 and path without leading `/` |
| apk | `/lib/apk/db/installed`: `P:` package, `V:` version, `F:` directory, `R:` file | `Z:` line after `R:`: `Q1` + base64 of SHA-1 |
| rpm | rpm database, read through Syft's catalog output | Per-file digest from the rpm header |
| Python | `*.dist-info/RECORD` | SHA-256 per file |
| Node | `node_modules/**/package.json` directory ownership | none |

**Matching rule.** A package-listed path and a file instance match if their **canonical paths** (§1) are equal. This avoids false `unknown` results on merged-`/usr` images, where the database may list `/bin/x` while the file lives at `/usr/bin/x`.

**Owner values.**

| Value | Condition |
|---|---|
| `distro` | Listed by dpkg, apk, or rpm, and integrity data matches (or no integrity data exists for that file) |
| `distro_modified` | Listed by dpkg, apk, or rpm, and integrity data does **not** match |
| `third_party` | Not distro; listed by a language package record |
| `unknown` | None of the above |

**Non-distro files** = `third_party` ∪ `unknown`. **Vendor-code proxy** (used where the plan says "internal") = `unknown`.

## 7. Image user and writability

The configured user is `config.User` from the image configuration; empty means `root` (uid 0). Names are resolved to uid/gid through `/etc/passwd` and `/etc/group` in the final filesystem; unresolvable names are recorded and treated as uid 65534.

A directory is **writable by the image user** if, from its tar header: the user is root; or `uid` matches and the owner write bit is set; or `gid` matches the user's primary group and the group write bit is set; or the other write bit is set. Supplementary groups are resolved from `/etc/group`.

## 8. Release series and identity

### 8.1 Release selection

For each stratum-C product, list tags in the registry, keep those that parse as semantic versions without pre-release suffix (no `-rc`, `-beta`, `-alpha`, `nightly`, `latest`, `edge`), sort by version, and take the **4 most recent** on the date of task T2. Consecutive releases in that list form the 3 release pairs. Tags are resolved to manifest digests on that date and never re-resolved.

### 8.2 Identity keys (M15)

| Key | Definition |
|---|---|
| K1 `path` | Canonical path |
| K2 `soname` | `DT_SONAME` value (shared libraries only) |
| K3 `stem` | Canonical path with a trailing version stripped: `\.so(\.\d+)+$` → `.so`, `-\d+(\.\d+)+` before the extension removed |
| K4 `package+stem` | Owning package name (without version) + K3 |
| K5 `build_id` | Build-id (expected to change on every rebuild; reported only as a sanity check) |

A key **matches** a file in release *n+1* if exactly one file in release *n* has the same key value; zero is `unmatched`, more than one is `ambiguous`.

- **Match rate** of a key = matched files in *n+1* ÷ files in *n+1* that exist in some form in *n* (union of matches over K1–K4).
- **False match** = a matched pair judged, on manual review, to be different programs (different owning package name, or different purpose evident from path and `DT_SONAME`). Review sample: 30 matched pairs per key, drawn as in §12.

**Primary matching for regression counting (pre-registered):** K1; if unmatched, K2; if still unmatched, K4. First match wins. H3.4 evaluates each key separately.

### 8.3 Regression

A **regression** is recorded for a matched pair (*a* in release *n*, *b* in release *n+1*) and a property *p* when all hold:

1. *p* applies to both *a* and *b*.
2. The fact for *p* is `certain` in both.
3. The value of *b* is **worse** than the value of *a* by this order:

| Property | Order (worse → better) |
|---|---|
| `pie` | absent → present |
| `nx_stack` | exec → missing → nx |
| `rwx_segment` | present → none |
| `textrel` | present → absent |
| `relro` | none → partial → full |
| `cet_ibt`, `cet_shstk`, `bti`, `pac` | absent → present |
| `search_path_risk` | number of risky entries increases |

`canary` and `fortify` changes are reported separately as **high/medium-confidence changes**; they are not regressions for H1.3.

A file present in *n+1* but not matched in *n* is **new**, never a regression. If it fails `standard`, it is counted as `new_fails_profile`.

A release pair **has a regression** if at least one regression is recorded on a **non-distro** file. Regressions on distro files are counted and reported but do not satisfy H1.3.

## 9. Cause attribution (M07)

Applied to every regression, and to every non-distro gap counted in H1.2. Rules in order; the first match wins; exactly one cause per case.

| # | Condition | Cause |
|---|---|---|
| 1 | File is `distro`/`distro_modified` in *b*, and the owning package version differs between *n* and *n+1* | H2.1 distro / base image |
| 2 | File is `distro` in *b*, package version unchanged, but the base-image identification (§9.1) changed | H2.1 distro / base image |
| 3 | File is non-distro, and `comment_compilers` or `go_version` differ between *a* and *b* | H2.2 toolchain |
| 4 | Gap only (not a regression): `language = go`, property `pie` absent | H2.4 language default |
| 5 | File is `third_party` and its package version differs, or the file is new and `third_party` | H2.3 prebuilt component |
| 6 | File is non-distro, both *a* and *b* have toolchain evidence (`comment_compilers` or `go_version`), and it is unchanged | H2.3 build configuration (inferred) |
| 7 | Otherwise | unknown |

Rule 6 is an inference by elimination and is labelled as such in every report.

### 9.1 Base-image identification

In order: the OCI annotation `org.opencontainers.image.base.digest` if present; otherwise the longest prefix of layer digests shared with a stratum-A image or a tagged release of a known base image; otherwise the set of distro package names and versions (a change in more than 50% of distro packages counts as a base change). The method used is recorded per image.

## 10. Effective marking (H3.6, M13–M14)

Only for executable kinds with a `PT_INTERP`, on the same architecture.

**Closure resolution, glibc model, inside the image root:**

1. For the executable, and then recursively for each loaded object, resolve each `DT_NEEDED` entry:
   - if it contains `/`, use it as a path;
   - otherwise search, in order: the object's `DT_RPATH` (only if the object has no `DT_RUNPATH`), then the executable's `DT_RPATH` (same condition); `LD_LIBRARY_PATH` **only** if set in the image configuration `Env`; the object's own `DT_RUNPATH` (never inherited by its dependencies); directories listed in `/etc/ld.so.conf` and its `include`d files; then `/lib/<multiarch>`, `/usr/lib/<multiarch>`, `/lib64` or `/lib`, `/usr/lib64` or `/usr/lib`.
   - `$ORIGIN` expands to the directory of the object containing the entry. `$LIB` and `$PLATFORM` are recorded and treated as unresolved in Phase 0.
2. The first existing in-scope ELF file with matching architecture wins.
3. Any unresolved `DT_NEEDED` marks the closure **incomplete**.

musl images use `/etc/ld-musl-<arch>.path` if present, otherwise `/lib:/usr/local/lib:/usr/lib`, with no `ld.so.cache` and no `DT_RUNPATH` inheritance differences **[verify M25]**.

**Effective loss.** For an executable declaring `cet_shstk` (x86_64) or `bti` (aarch64): *loses* the marking if any object in a **complete** closure lacks the same marking. Incomplete closures are excluded from both numerator and denominator and counted separately. Libraries loaded only at runtime by name (not in `DT_NEEDED`) are out of scope for Phase 0 and noted as a limitation.

## 11. Measurements that are not binary analysis

### 11.1 Release-note check (M20, H1.4)

For each release pair with a regression, search: the GitHub release body of *n+1*; `CHANGELOG*` / `NEWS*` / `RELEASE*` files at the *n+1* tag; issues and pull requests closed between the two release dates. Keywords (case-insensitive): `pie`, `position independent`, `relro`, `bindnow`, `-z now`, `stack protector`, `stack-protector`, `fortify`, `hardening`, `cet`, `shadow stack`, `bti`, `branch protection`, `buildmode=pie`, `textrel`, `execstack`, `rpath`, `runpath`, plus the file name and owning package of the regressed file.
A regression is **mentioned** if a hit refers to the regressed property or component in a build-hardening context, confirmed by manual reading and logged with the URL.

### 11.2 Timed walkthrough (M22, H1.5)

- **Allowed tools:** `docker` or `crane`, `checksec`, `readelf`, a shell. No scripts written in advance.
- **Start:** image reference known, nothing pulled. **Stop:** a written answer (present / absent / unknown) for PIE, RELRO, canary, and CET for every in-scope ELF file — or 30 minutes elapsed, whichever comes first; if 30 minutes elapse, continue until complete or 90 minutes.
- **Recorded:** timestamps in the journal at start, first answer, 30 minutes, and completion; coverage = files answered ÷ in-scope ELF count from M01.

### 11.3 CI configuration survey (M21, H2.5)

At the commit of the latest sampled release tag, inspect `.github/workflows/*`, `.gitlab-ci.yml`, `Jenkinsfile*`, `.circleci/config.yml`, `azure-pipelines.yml`, `Makefile` targets named `release*`/`check*`. A product **has a hardening check** if a CI step runs one of `checksec`, `hardening-check`, `annocheck`, `binskim`, `scanelf`, or a `readelf` invocation whose output is tested for `GNU_RELRO`, `BIND_NOW`, `PIE`, or `GNU_STACK`, **on build output**. Every hit is confirmed manually and logged.

### 11.4 Tool comparison (M12, H2.6)

- **Tools and versions** are pinned in the toolbox image and recorded in the run log.
- **Normalization:** each tool's output is mapped to this protocol's values by a mapping table written from the tools' documentation **before** the comparison is run, and frozen with this protocol at T0 (appendix A, completed in T1). Outputs such as "N/A", "unknown", or a missing field map to `undeterminable`.
- **Disagreement** = two tools give different normalized values where both give a value. Abstentions are counted separately, never as disagreements.
- **"Most-used tool"** = checksec, by pre-registration.
- **Named cases:** `language = go`; `libc = musl` (FORTIFY); `elf_kind ∈ {static, static_pie}` and `stripped`; `elf_kind` rule 5 vs. rule 6 (PIE vs. shared library).

## 12. Sampling and adjudication

- **Random seed:** `20261001`, used by a documented pseudo-random generator (Python `random.Random(seed)`) for every sample in Phase 0.
- **Procedure:** list the population sorted by (image digest, canonical path, property); shuffle with the seed; take the first *k*.
- **Sizes:** 30 per check (or the whole population if smaller); 50 for M12 disagreements, stratified equally across the four named cases.
- **Manual check:** the adjudicator runs the commands listed per property (appendix B), records the deciding output lines, and fills the adjudication log ([plan §12.6](phase-0-plan.md#126-adjudication-log)).
- **Acceptance:** if manual and automated values disagree in more than 5% of a sample, the extractor is fixed, the run is repeated, and a new sample is drawn with seed + 1.

## 13. Statistics and verdict rules

- Every proportion is reported as **count / denominator** with a **95% Wilson score interval**.
- The verdict is decided on the **point estimate** against the pre-registered threshold.
- If the interval contains the threshold, the verdict is still given but marked **fragile** in the findings, and no product decision may rest on a fragile verdict alone.
- Denominators always include excluded items as a separate count: **total / analyzed / excluded**.
- A hypothesis whose analyzed denominator is below 10 is **Inconclusive** regardless of the point estimate.

## 14. Threshold definitions (binding for plan §4)

| Hypothesis | Numerator | Denominator | Threshold |
|---|---|---|---|
| H1.2 | Products where ≥ 10% of non-distro in-scope files fail ≥ 1 of: `pie` (executables; C/C++/Rust only), `nx_stack` ≠ nx, `rwx_segment` present, `textrel` present, `relro` = none, `search_path_risk` risky — all `certain` | 10 stratum-B products (latest release) | ≥ 3 |
| H1.3 | Release pairs with ≥ 1 regression on a non-distro file (§8.3) | 30 release pairs | ≥ 6 |
| H1.4 | Regressions from H1.3 not mentioned (§11.1) | All regressions from H1.3 | ≥ 70% |
| H1.5 | Walkthrough duration; coverage at 30 min (§11.2) | — | > 60 min, or < 20% coverage |
| 2A | Cases per cause (§9) | All attributed cases excluding `unknown` | dominant ≥ 50%, contributing ≥ 15% |
| H2.5 | Products with a hardening check (§11.3) | 10 stratum-B products | ≤ 2 |
| H2.6 | Properties with disagreement ≥ 5% **and** checksec wrong in ≥ 50% of adjudicated named-case disagreements | Properties compared | ≥ 1 |
| H2.7 | Non-distro files with owner `unknown` | Non-distro in-scope files | ≥ 20% |
| H2.8 | Median in-scope ELF count | 10 stratum-B products (latest release) | ≥ 200 |
| H3.1 | Per `strict` property: files decidable at `certain` or `high` | Non-distro in-scope files where the property applies | ≥ 90% each; and files with `comment_compilers` non-empty ÷ non-distro files ≥ 50% |
| H3.2 | Regressions attributed to rules 1–5 (rule 6 and unknown excluded) | All regressions | ≥ 60% |
| H3.3 | Wrong `block` decisions on manual review (wrong fact or wrong match) | All `block` decisions in the simulation | < 2%; and median blocks per pair ≤ 5 |
| H3.4 | Per key: match rate and false-match rate (§8.2) | As defined in §8.2 | best key ≥ 95% match and ≤ 1% false |
| H3.6 | Executables losing SHSTK/BTI effectively (§10) | Executables declaring it with a complete closure | ≥ 5% |

**Policy simulation for H3.3.** Profile `standard` as in [ADR 0003](../adr/0003-policy-profiles.md), restricted to `certain` facts; gate matrix `pr` defaults from [ADR 0004](../adr/0004-gate-rules-and-matrix.md), with owner `unknown` treated as `internal` (vendor-code proxy); reference snapshot = release *n*; new files evaluated against `strict`, with [ADR 0007](../adr/0007-strict-toolchain-adjustment.md) adjustments applied. A `block` is **wrong** only if the fact is wrong or the files are not the same program — not if the reviewer disagrees with the policy.

## Appendix A. Tool output mapping

To be completed in task T1 from each tool's documentation, **before** any comparison is run, and frozen at T0. One table per tool: tool field → tool value → protocol property → protocol value.

## Appendix B. Manual adjudication commands

| Property | Command | Deciding lines |
|---|---|---|
| `elf_kind`, `pie` | `readelf -hlW <f>`; `readelf -dW <f>` | `Type:`; `INTERP`; `FLAGS_1` containing `PIE`; `SONAME` |
| `nx_stack`, `rwx_segment` | `readelf -lW <f>` | `GNU_STACK` flags; `LOAD` flags `RWE` |
| `textrel` | `readelf -dW <f>` | `TEXTREL`, or `FLAGS` containing `TEXTREL` |
| `relro` | `readelf -lW <f>`; `readelf -dW <f>` | `GNU_RELRO`; `BIND_NOW`, `FLAGS` `BIND_NOW`, `FLAGS_1` `NOW` |
| `cet_*`, `bti`, `pac` | `readelf -nW <f>` | `x86 feature: IBT, SHSTK` / `AArch64 feature: BTI, PAC` |
| `rpath`, `runpath` | `readelf -dW <f>` | `RPATH`, `RUNPATH` |
| `canary`, `fortify` | `readelf --dyn-syms -W <f>` | `UND` entries `__stack_chk_*`, `__*_chk` |
| `build_id` | `readelf -nW <f>` | `Build ID:` |
| `comment_compilers` | `readelf -p .comment <f>` | Compiler strings |
