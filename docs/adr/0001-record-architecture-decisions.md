# 0001. Record architecture decisions

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-09-23 |
| **Deciders** | @Lewall-theart |

## Context

Decisions about architecture, dependencies, and security trade-offs are made throughout a product's life. Without a record, the reasoning is lost: new contributors cannot tell whether a choice was deliberate, and old debates are reopened without new information.

## Options considered

1. **No formal record** — rely on PR descriptions and memory.
2. **A single design document** kept up to date — easy to find, but history is overwritten.
3. **Architecture Decision Records** — one short, immutable file per decision, stored with the code.

## Decision

We will record every significant decision as an ADR in `docs/adr/`, using [0000-template.md](0000-template.md), numbered sequentially. An ADR is never edited after it is accepted; a new ADR supersedes it instead.

A decision is significant if it is hard to reverse, affects security posture, adds a runtime dependency, or changes a public interface.

## Consequences

**Positive**

- Reviewers and future maintainers can see why the system looks the way it does.
- Security trade-offs are explicit and reviewable.

**Negative**

- A small amount of writing overhead per decision.

**Follow-up actions**

- [x] Add the ADR template to the repository template.
