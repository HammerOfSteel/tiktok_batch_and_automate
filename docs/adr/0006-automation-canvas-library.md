# ADR-0006: Svelte Flow for the automation canvas

- **Status:** Proposed, pending spike `S-02`
- **Date:** 2026-08-16
- **Related:** [features/automation-engine.md](../features/automation-engine.md), task `0.1.2`

## Context

The automation workspace needs a node based canvas with the feel of n8n: a node palette, drag onto the canvas,
connect ports, pan, zoom, a minimap, multi select, undo and redo, and custom node rendering with our own
components.

Building this is deceptively large. Edge routing, port hit testing, zoom aware pointer maths, selection
rectangles and viewport culling are each a project.

## Options considered

**A. Svelte Flow (`@xyflow/svelte`).**
Pros: the Svelte port of React Flow by the same team, purpose built for exactly this. Custom nodes are Svelte
components, so shadcn-svelte components can be used directly. Handles viewport, edges, minimap, controls and
selection. Graph state is plain nodes and edges arrays, which matches our JSON storage format almost exactly.
MIT licensed.
Cons: a smaller community than React Flow, so fewer examples. Svelte 5 compatibility must be verified. Some
advanced features are in the paid React Flow Pro tier, so those need checking against our requirements.

**B. Build it ourselves on SVG or Canvas.**
Pros: exactly the features we want, no dependency, full control of performance.
Cons: weeks of work reimplementing solved problems, and the accessibility and touch story alone is substantial.
Directly contradicts the "work smarter, not harder" principle.

**C. Embed a React island with React Flow.**
Pros: the most mature library available.
Cons: two frameworks in one bundle, an awkward interop boundary for state, and a permanent maintenance tax.
Only justified if no Svelte option works at all.

**D. Wrap a generic diagramming library (JointJS, mxGraph, Rete.js).**
Pros: mature and feature rich.
Cons: heavy, framework agnostic APIs that fight Svelte reactivity, licensing needs checking, and custom node
rendering is far less pleasant than "it is just a component".

## Decision (proposed)

Option A. Svelte Flow, subject to spike `S-02`, which must produce a throwaway prototype demonstrating:

1. Three custom node types rendered as Svelte components with shadcn-svelte inside them.
2. Drag from the palette, connect ports, and reject invalid connections.
3. Save the graph to JSON, reload it, and get an identical graph. Round tripping must be lossless.
4. Undo and redo.
5. Acceptable performance at 100 nodes.
6. Keyboard accessibility for creating and connecting nodes, or a documented alternative path, since a canvas
   that is mouse only excludes users.
7. Licence confirmed for our use, with no required paid tier for anything in
   [features/automation-engine.md](../features/automation-engine.md).

If any of 1 to 5 fails, reconsider between option B for a deliberately simpler canvas and option C.

## Consequences

**Easier:** the hard parts of canvas interaction come for free. Our node components stay ordinary Svelte
components. The stored graph format is close to the library's own model, so persistence is thin.

**Harder:** a dependency on a comparatively young package for a headline feature. Our stored format must stay
**ours**, mapped to and from the library's model at the boundary, so a library change never forces a data
migration.

**Accepted costs:** a mapping layer between our graph JSON and the library's node and edge model. That is a
deliberate cost, not an accident.

**Revisit if:** the spike fails a requirement, the package stops being maintained, or accessibility proves
unworkable.
