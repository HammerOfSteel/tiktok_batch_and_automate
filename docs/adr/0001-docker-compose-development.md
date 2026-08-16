# ADR-0001: Docker Compose as the only development environment

- **Status:** Accepted
- **Date:** 2026-08-16
- **Related:** [development.md](../development.md), task `0.3.3`

## Context

The stack has several moving parts: a SvelteKit frontend, a NestJS API, a worker process, PostgreSQL, Redis,
object storage and a mail sink. A "install Node, install Postgres, install Redis, run four terminals" onboarding
guarantees that every machine drifts and that "works on mine" becomes the default state.

The project also has an explicit principle: the whole stack, frontend included, runs in Docker Compose so local
development is easy.

## Options considered

**A. Docker Compose for everything, including the frontend.**
Pros: one command, identical on every machine, no host runtime required, CI and local run the same images,
service boot order is explicit and healthcheck driven.
Cons: frontend hot reload needs careful bind mount and file watching configuration, container filesystem
performance on macOS is worse than native, `node_modules` needs a named volume to avoid host and container
fighting over binaries.

**B. Compose for infrastructure only, run web and API on the host.**
Pros: the fastest possible frontend hot reload, direct debugger attachment with no configuration.
Cons: every contributor needs the right Node version, "it works on my machine" returns, CI and local diverge,
onboarding becomes a list of prerequisites.

**C. Devcontainers.**
Pros: reproducible, editor integrated, popular.
Cons: ties the workflow to VS Code, adds a layer on top of Compose that we would still need to maintain, and
does not remove the need for the Compose file.

## Decision

Option A. Everything runs in Docker Compose, frontend included. A contributor needs only Docker and git.

Devcontainers are not excluded later, but they would sit on top of the Compose stack rather than replace it.

## Consequences

**Easier:** onboarding is a clone and one command. Environment parity between local, CI and production is real
rather than aspirational. New services are added in one file. Boot order is deterministic through healthchecks.

**Harder:** hot reload needs deliberate configuration (polling where inotify does not cross the boundary,
named volume for `node_modules`, correct file watching options). Adding a dependency requires an image rebuild.
Debugging needs the inspector port exposed and a launch configuration.

**Accepted costs:** slower filesystem on macOS, a rebuild step when dependencies change.

**Revisit if:** frontend iteration speed becomes painful enough to hurt productivity measurably, in which case a
documented "host frontend" escape hatch may be added, without ever becoming the default path.
