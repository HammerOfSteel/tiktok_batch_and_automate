# ADR-0002: SvelteKit with shadcn-svelte for the frontend

- **Status:** Accepted
- **Date:** 2026-08-16
- **Related:** [architecture.md](../architecture.md), [features/dashboard.md](../features/dashboard.md)

## Context

The frontend is the product for most of the first phase: a dense card grid with multi select, a taxonomy tree, a
job centre and a node based canvas. It needs to be fast with hundreds of cards on screen, accessible, and quick
to build with a component library rather than hand rolled primitives.

The project brief already names [shadcn-svelte](https://github.com/huntabyte/shadcn-svelte) as the UI base,
which implies Svelte.

## Options considered

**A. SvelteKit + Svelte 5 + Tailwind + shadcn-svelte.**
Pros: matches the stated intent, small bundles, excellent runtime performance for dense grids, runes give
straightforward fine grained reactivity for selection state, components are vendored into the repository so they
can be adapted rather than fought, first party routing and load functions, good Playwright story.
Cons: a smaller ecosystem than React, fewer ready made complex components (data grids, canvases), Svelte 5 is
comparatively young so some libraries lag.

**B. Next.js + shadcn/ui.**
Pros: the largest ecosystem, the most mature component and canvas libraries, easiest hiring.
Cons: contradicts the stated preference, heavier runtime for a grid heavy UI, React Flow would be the canvas
choice and is excellent, but the rest of the stack would be a bigger bundle for no product benefit.

**C. Nuxt or SolidStart.**
Pros: each has merits.
Cons: neither is preferred by the project, and neither offers something the other two do not.

## Decision

Option A. SvelteKit 2 with Svelte 5, TypeScript in strict mode, Tailwind, and shadcn-svelte for components.

Frontend structure follows the layering in [architecture.md](../architecture.md): generated API client in
`lib/api`, pure logic in `lib/domain`, presentational components in `lib/components`, feature slices in
`lib/features`.

## Consequences

**Easier:** dense grids stay smooth without virtualisation tricks. Selection state is simple with runes.
Vendored components can be adapted freely. Bundle size stays small.

**Harder:** fewer off the shelf complex components, so some things get built. Library support for Svelte 5 must
be checked before adopting anything, which is exactly what spike `S-02` does for the canvas.

**Accepted costs:** we may write a component that React would have given us for free.

**Revisit if:** spike `S-02` finds no viable Svelte canvas library and building one would cost more than the
whole frontend, in which case the canvas alone could be isolated, though a framework change is far more likely
to be the wrong answer than the right one.
