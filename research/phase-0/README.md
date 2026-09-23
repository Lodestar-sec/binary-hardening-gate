# Phase 0 research workspace

Implements the [research plan](../../docs/research/phase-0-plan.md) and [measurement protocol](../../docs/research/phase-0-protocol.md). This is research tooling, not the product engine.

## Layout

| Path | Content | Committed |
|---|---|---|
| `toolbox/Dockerfile` | Pinned analysis tools (T1) | Yes |
| `sample.yaml` | Images with pinned digests (T2) | Yes |
| `scripts/` | Collection and analysis scripts (T3+) | Yes |
| `runs/<run-id>/` | Run log, events, exclusions, tool stderr (plan §12.2, §12.4) | Yes |
| `datasets/<version>/` | Derived JSONL + `MANIFEST.json` (plan §12.3) | Yes |
| `analysis/` | Analysis logs per hypothesis (plan §12.7) | Yes |
| `journal.md`, `deviations.md`, `decisions.md` | Human logs (plan §12.8) | Yes |
| `sources.csv`, `adjudication.csv` | Source and adjudication logs (plan §12.5, §12.6) | Yes |
| `work/` | Exported image filesystems | **No — never commit binaries** |

## Safety rules

1. Analyzed binaries are **never executed**. Do not run `ldd` on them; use `lddtree` / `readelf`.
2. Images are exported with `crane`, never started with `docker run <image>`.
3. Analysis runs with **no network**, the exported filesystem mounted **read-only**, and CPU/memory limits.
4. `work/` stays local; it is git-ignored.

## Two container modes

The same toolbox image is used in two modes, so the step that touches the network never touches extracted binaries at the same time.

**Fetch** (network on, writes `work/`):

```bash
docker run --rm -v "$PWD/work:/work" lodestar/research-toolbox \
  crane export <ref>@<digest> /work/<image-id>.tar
```

**Analyze** (network off, read-only input):

```bash
docker run --rm --network none --read-only --tmpfs /tmp \
  --cpus 2 --memory 4g --pids-limit 256 \
  -v "$PWD/work:/work:ro" -v "$PWD/runs:/runs" -v "$PWD/datasets:/datasets" \
  lodestar/research-toolbox python3 /opt/scripts/collect.py --run-id <run-id>
```

`crane export` yields the final filesystem (whiteouts applied). `layer_index` and base-image identification (protocol §9.1) additionally need the per-layer view: `crane pull --format=oci <ref>@<digest> /work/<image-id>.oci`.

## Session checklist

1. Add a journal entry (date, hours, task).
2. Every run gets a new `runs/<run-id>/`; never overwrite.
3. Log every exclusion, source, adjudication, deviation, and decision as it happens.
4. Commit logs at the end of the session (signed).
