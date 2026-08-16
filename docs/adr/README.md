# Architecture decision records

Each significant technical decision gets a numbered, immutable record. If a decision changes, write a new ADR
that supersedes the old one. Never edit history, the reasoning at the time is the valuable part.

## Status meanings

| Status | Meaning |
| --- | --- |
| `Proposed` | Written down, not yet committed to. Usually pending a spike |
| `Accepted` | In force. The codebase follows it |
| `Superseded by ADR-XXXX` | No longer in force, kept for the reasoning |
| `Deprecated` | No longer in force, with no replacement |

## Index

| ADR | Title | Status |
| --- | --- | --- |
| [0001](0001-docker-compose-development.md) | Docker Compose as the only development environment | Accepted |
| [0002](0002-frontend-stack.md) | SvelteKit with shadcn-svelte for the frontend | Accepted |
| [0003](0003-backend-stack.md) | NestJS, Prisma and PostgreSQL for the backend | Proposed |
| [0004](0004-authentication.md) | Self hosted sessions plus TikTok OAuth for linking | Proposed |
| [0005](0005-job-queue.md) | BullMQ on Redis for jobs and scheduling | Proposed |
| [0006](0006-automation-canvas-library.md) | Svelte Flow for the automation canvas | Proposed |
| [0007](0007-seeded-adapter-over-mock-server.md) | A seeded adapter behind a port, not a mock server | Accepted |

## Template

```markdown
# ADR-XXXX: Title

- **Status:** Proposed | Accepted | Superseded by ADR-YYYY
- **Date:** YYYY-MM-DD
- **Deciders:** who
- **Related:** spike, task or ADR references

## Context
What forces are at play. What problem needs solving. What constraints exist.

## Options considered
Each option with honest pros and cons. At least two, otherwise this is not a decision.

## Decision
What we chose, stated plainly.

## Consequences
What becomes easier. What becomes harder. What we accept as a cost. What would make us revisit this.
```
