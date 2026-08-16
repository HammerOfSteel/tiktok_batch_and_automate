# Documentation

Start here.

## Read in this order if you are new

1. [overview.md](overview.md) - what the product is, who it is for, what is out of scope
2. [TODO.md](TODO.md) - the phased plan with tasks, gates and commit points
3. [architecture.md](architecture.md) - layering, modules and dependency rules
4. [development.md](development.md) - get the stack running

## Reference

| Document | Contents |
| --- | --- |
| [overview.md](overview.md) | Product, personas, goals, non goals, success criteria, risks |
| [TODO.md](TODO.md) | Phases 0 to 6, tasks, subtasks, quality gates, commit and push points |
| [architecture.md](architecture.md) | Onion layers, module boundaries, ports and adapters, request lifecycle |
| [api-guidelines.md](api-guidelines.md) | Zalando derived REST conventions, errors, pagination, idempotency |
| [data-model.md](data-model.md) | Aggregates, tables, indexes, local versus remote ownership |
| [testing-strategy.md](testing-strategy.md) | Levels, tools, coverage thresholds, what we do not test |
| [development.md](development.md) | Compose stack, commands, debugging, common problems |
| [design-system.md](design-system.md) | Visual registers, component allowlist, motion rules, performance budgets |
| [security.md](security.md) | Threat model, auth, tokens, destructive action safety, OWASP mapping |
| [tiktok-integration.md](tiktok-integration.md) | Capability matrix, prior art, client design, sync and publish design, compliance |
| [glossary.md](glossary.md) | Ubiquitous language, and the words we deliberately avoid |

## Feature specifications

| Document | Contents |
| --- | --- |
| [features/dashboard.md](features/dashboard.md) | Card grid, sorting, filtering, multi select, empty states |
| [features/batch-actions.md](features/batch-actions.md) | Job model, selections, safety mechanisms, execution, reporting |
| [features/automation-engine.md](features/automation-engine.md) | Node catalogue, execution semantics, builder, graph format |
| [features/tagging.md](features/tagging.md) | Categories, tags, assignment surfaces, API |

## Decisions

[adr/](adr/) holds the architecture decision records, with an index and the template.

## Conventions for these documents

- British English, no em-dashes, active voice, plain language.
- Tables and short sections over long prose.
- Every command shown must actually run.
- Anything unverified is labelled `UNVERIFIED` rather than stated as fact.
- Documentation is updated in the same commit as the change it describes.
