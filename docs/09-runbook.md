# 09 · Runbook: <product-name>

<!-- TEMPLATE: procedures are exercised in the ephemeral and release-verification clusters.
     Keep every procedure executable against a fresh `helm install` of the released chart. -->

## Service overview

| | |
|---|---|
| **What it does** | |
| **Owners** | @<github-handle> |
| **Dependencies** | |
| **Dashboards** | |
| **Logs** | |

## Service level objectives

| SLI | SLO | Window |
|---|---|---|
| Availability (successful requests / total) | 99.5% | 30 days |
| Latency (p95) | < 500 ms | 30 days |

## Alerts

Every alert must link to a section below. An alert without a procedure is noise.

| Alert | Severity | Meaning | Procedure |
|---|---|---|---|
| <AlertName> | Page / Ticket | | [link](#procedure-alertname) |

## Procedures

### Procedure: <AlertName>

1. **Confirm** —
2. **Mitigate** —
3. **Resolve** —
4. **Follow up** — open a postmortem issue if user impact exceeded the SLO budget.

### Releasing

1.
2.

### Rolling back a release

1.
2.

### Rotating secrets

1.
2.

## Escalation

| Level | Who | When |
|---|---|---|
| 1 | Maintainer on duty | Any alert |
| 2 | @<github-handle> | No mitigation within 1 hour |
