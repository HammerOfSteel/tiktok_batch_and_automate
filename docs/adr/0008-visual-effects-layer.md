# ADR-0008: Aceternity UI Svelte as a bounded expressive layer

- **Status:** Proposed, pending spike `S-05`
- **Date:** 2026-08-16
- **Related:** [design-system.md](../design-system.md), [ADR-0002](0002-frontend-stack.md), task `0.1.7`

## Context

shadcn-svelte gives excellent, quiet primitives. It does not give a landing page that makes anyone stop
scrolling, and it does not give the small moments of life that make an application feel considered rather than
generated.

[Aceternity UI Svelte](https://aceternity.sveltekit.io/components) is a Svelte port of Aceternity UI: animated
gradient backgrounds, beams, spotlights, sparkles, bento grids, reveal effects. Roughly forty components,
copy-paste in the same style as shadcn.

The risk is obvious. Effect libraries are extremely easy to overuse, and a data dense dashboard smothered in
animation is worse than a plain one. The other risk is performance: several of these components run per frame
work, and our dashboard renders hundreds of cards.

## Options considered

**A. Adopt it, bounded by an explicit allowlist and a budget.**
Pros: a genuinely striking landing page and auth flow for very little effort. Copy-paste means no runtime
dependency and full freedom to adapt. Same Tailwind foundation as shadcn-svelte, so tokens and theming are
shared. We choose exactly which components exist in the codebase.
Cons: ported showcase components are written for looks, so accessibility and reduced motion support must be
audited and often added by us. Some components assume Framer Motion idioms from the React original, and the
Svelte equivalents need checking. Licence needs verifying for both the original and the port.

**B. Do not adopt. Build the few effects we want by hand with CSS and Tailwind.**
Pros: total control, minimal code, nothing to audit but our own work.
Cons: an animated gradient mesh or a beams background is a genuine time sink to build well, and it is exactly
the kind of solved problem the project's "work smarter, not harder" principle says to reuse.

**C. Adopt it broadly, including in the application surfaces.**
Pros: a consistent, distinctive feel everywhere.
Cons: directly threatens the dashboard's performance budget and its core interaction, which is precise multi
select over a dense grid. 3D card effects on 500 video cards is a straightforward way to make the product feel
slow.

## Decision (proposed)

Option A, with hard boundaries:

1. **Two registers.** Expressive on the landing, auth, onboarding and empty states. Utilitarian in the
   dashboard, job centre, canvas and settings. Defined in [design-system.md](../design-system.md).
2. **An allowlist.** Only vetted components are vendored in. Anything else needs a pull request justification.
3. **Copy in, own it.** Components live in `lib/components/effects`, are ours from that moment, and are audited
   for accessibility and `prefers-reduced-motion` before first use.
4. **Budgets are gates.** Bundle size, idle CPU, scroll frame rate and LCP limits are enforced at the Phase 1
   and Phase 6 gates. A component that misses a budget does not ship.
5. **One animated background per page.** Not a guideline, a rule.
6. **Nothing on the video card.** The card is the hottest component in the product and stays plain.

Spike `S-05` (task `0.1.7`) verifies before this becomes Accepted:

- Licence of both the original Aceternity UI and the Svelte port, confirmed as compatible with our use.
- Svelte 5 compatibility of the components we actually want.
- Which motion dependency the port pulls in, and its size.
- Measured idle CPU and bundle cost of one animated background component.
- That `prefers-reduced-motion` can be honoured, in the component or by our adaptation.

## Consequences

**Easier:** a landing page and auth flow that look deliberate, with a day of work rather than a week. Shared
Tailwind tokens mean the two libraries theme together. No runtime dependency to keep updated.

**Harder:** every vendored effect is our maintenance and our accessibility problem. Copy-paste means no
upstream fixes, so an effect we take is an effect we own. Discipline is required, and the allowlist plus the
budgets exist precisely because taste alone will not hold the line under deadline pressure.

**Accepted costs:** a small amount of duplicated component code, and an audit step before first use of each
effect.

**Revisit if:** the spike finds a licence problem or Svelte 5 incompatibility, or if the effects layer starts
appearing in the application surfaces, which would mean the boundary failed and needs to be enforced by lint
rather than by convention.
