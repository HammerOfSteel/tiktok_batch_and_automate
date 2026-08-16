# Contributing

## Before you start

1. Read [docs/overview.md](docs/overview.md) for what we are building and why.
2. Read [docs/architecture.md](docs/architecture.md) for the layering rules. They are enforced by lint, not by
   review alone.
3. Find your task in [docs/TODO.md](docs/TODO.md). Work is planned there, in order, with gates.
4. Get the stack running with [docs/development.md](docs/development.md).

If what you want to do is not in the plan, add it to the backlog section of `TODO.md` and raise it, rather than
opening a pull request that quietly expands scope.

## Workflow

```bash
git switch -c feat/1.2.4-multi-select     # branch name carries the task number
# work, in small commits
pnpm lint && pnpm typecheck && pnpm test
git push -u origin feat/1.2.4-multi-select
# open a pull request
```

### The rhythm

| When | Do this |
| --- | --- |
| A task from `TODO.md` is complete | **Commit.** One task, one commit, the message is given in the task |
| A phase gate passes | **Push and tag.** Not before the gate passes |
| A gate fails | Fix it before starting the next phase. Never carry a failed gate forward |

Committing per task keeps history readable and makes a bad change trivially revertible. Pushing per phase means
what lands on the main branch has passed a gate.

## Branches

| Pattern | For |
| --- | --- |
| `feat/<task>-<slug>` | A new capability |
| `fix/<slug>` | A bug fix |
| `refactor/<slug>` | Behaviour preserving change |
| `docs/<slug>` | Documentation only |
| `chore/<slug>` | Tooling, dependencies, configuration |
| `spike/<id>-<slug>` | Research. **Never merged.** The output is an ADR |

`main` is always releasable. It is protected: CI must pass, and a review is required.

## Commits

Conventional Commits, enforced by `commitlint`.

```
<type>(<scope>): <subject>

[body: why, not what]

[footer: refs #issue, BREAKING CHANGE: ...]
```

Types: `feat`, `fix`, `refactor`, `perf`, `test`, `docs`, `chore`, `ci`, `build`, `style`.
Scopes: `domain`, `api`, `web`, `worker`, `infra`, `contracts`, `docker`, `automation`, `repo`, `ops`.

```
feat(web): add multi select with selection bar

Implements task 1.2.4. Selection is represented as either explicit ids or
a filter reference so that "select all 2000 matching" does not build a
2000 item URL.
```

Rules:

- Subject in the imperative, lower case, no full stop, under 72 characters.
- The body explains **why**. The diff already shows what.
- One logical change per commit. If the subject needs an "and", split it.
- Never commit commented out code, a stray `console.log`, a secret or a `.env` file.
- Tests and documentation for a change go in the **same commit** as the change.

## Pull requests

Keep them small. A 200 line pull request gets a real review, a 2,000 line one gets a rubber stamp.

The description states:

1. Which task from `TODO.md` this is.
2. What changed and why.
3. How it was tested, with the commands run.
4. Screenshots or a recording, for any UI change.
5. Anything a reviewer should look at especially closely.

Merge requires: CI green, one approval, no unresolved conversations, and the task's own tests present.

We squash merge feature branches. The squashed message keeps the task reference.

## Code standards

### General

- TypeScript strict mode. `any` needs a comment justifying it, and there is almost never a justification.
- No dead code, no commented out blocks. Git remembers, that is its job.
- Comments explain **why**, never what. If the code needs a comment to explain what it does, rewrite the code.
- Names come from [docs/glossary.md](docs/glossary.md). If you need a new term, add it there in the same
  pull request.

### Backend

- The domain layer imports nothing from a framework, a database or an HTTP library. Lint enforces it.
- Business rules live in the domain, never in a controller and never in a repository.
- Expected failures return a `Result`. Exceptions are for bugs and infrastructure faults.
- Side effects (clock, randomness, id generation, IO) are injected, never called inline.
- Every endpoint exists in the OpenAPI spec **before** it exists in code.

### Frontend

- No `fetch` outside `lib/api`.
- No fixture data outside tests. This is checked at the Phase 1 gate.
- Filter, sort and page state lives in the URL.
- Components in `lib/components` are presentational. Anything that knows about videos lives in `lib/features`.
- Every interactive element is reachable by keyboard and has an accessible name.

### Tests

See [docs/testing-strategy.md](docs/testing-strategy.md). The short version:

- Tests are part of the change, not a follow up.
- Test names state the behaviour, not the method.
- No test depends on another test, or on test order.
- A flaky test is a bug. Fix it or delete it. Never wrap it in a retry.

## Documentation

Documentation is part of done.

| Change | Update |
| --- | --- |
| New or changed endpoint | The OpenAPI spec, plus [docs/api-guidelines.md](docs/api-guidelines.md) if a convention changes |
| New entity or field | [docs/data-model.md](docs/data-model.md) |
| New module or boundary | [docs/architecture.md](docs/architecture.md) |
| Feature behaviour | The relevant file in [docs/features/](docs/features/) |
| A technical decision | A new ADR in [docs/adr/](docs/adr/) |
| A new term | [docs/glossary.md](docs/glossary.md) |
| Task complete | Tick the box in [docs/TODO.md](docs/TODO.md) |

Style: British English, no em-dashes, plain language, active voice. Tables and short sections over long prose.
Every command shown must actually run.

## Decisions

A significant technical choice gets an ADR. Copy the template from [docs/adr/README.md](docs/adr/README.md),
give it the next number, and open it as a pull request so the decision is discussed before the code exists.

ADRs are immutable. Changing your mind means writing a new one that supersedes the old.

## Security

Never commit a secret. If you do, rotate it first and rewrite history second, and assume it is compromised.

Do not report a vulnerability in a public issue. See [docs/security.md](docs/security.md).

Any change touching authentication, authorization, token handling or a destructive action needs an explicit
review against the checklist in [docs/security.md](docs/security.md), noted in the pull request.
