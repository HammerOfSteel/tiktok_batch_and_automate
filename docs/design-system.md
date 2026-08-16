# Design system and motion

Two visual registers, one component foundation.

| Register | Where | Character |
| --- | --- | --- |
| **Expressive** | Landing page, login, onboarding, empty states, celebratory moments | Animated gradients, beams, spotlights, sparkles. Alive, memorable |
| **Utilitarian** | Dashboard, job centre, automation canvas, settings | Calm, dense, fast. Motion only when it communicates something |

The dashboard is a tool people use for hours. The landing page is a page people see for ten seconds. Treating
them the same is how you end up with either a boring product or an exhausting one.

## Foundations

| Layer | Source | Role |
| --- | --- | --- |
| Primitives | [shadcn-svelte](https://github.com/huntabyte/shadcn-svelte) | Buttons, inputs, dialogs, tables, menus. The everyday 95 percent |
| Expressive components | [Aceternity UI Svelte](https://aceternity.sveltekit.io/components) | Backgrounds, reveals, hero effects. The marketing and moment 5 percent |
| Tokens | Tailwind theme | Colour, spacing, radius, typography, shadow, motion duration |
| Icons | Lucide | One icon set, no mixing |

Both libraries are **copy in, not depend on**. Components are vendored into `lib/components`, so they are our
code and can be adapted, trimmed and made accessible. Record any non trivial adaptation in a one line comment
saying why.

## Component allowlist

Vetted before use, per [ADR-0008](adr/0008-visual-effects-layer.md). Adding anything not on this list requires
a note in the pull request describing where it is used and what it costs.

### Approved for the landing and auth surfaces

| Component | Use |
| --- | --- |
| Background Gradient / Gradient Animation | Hero background, one per page, subtle |
| Background Beams | Alternative hero background. Never on the same page as another animated background |
| Spotlight | Draw attention to the primary call to action, once |
| Lamp Effect | Section header on the landing page |
| Text Generate Effect / Typewriter | The headline only. Never on body copy, never on anything a user must read to proceed |
| Bento Grid | Feature showcase |
| Card Hover Effect | Feature or pricing cards |
| Infinite Moving Cards | Testimonials, if we ever have any |
| Sparkles | One accent moment. Not a page wide effect |
| Signup Form | Auth screens, layered on shadcn inputs |
| Moving Border | The single primary call to action button |

### Approved for the application surfaces

| Component | Use | Constraint |
| --- | --- | --- |
| Grid and Dot Backgrounds | Automation canvas background, empty states | Static, no animation |
| Animated Tooltip | Contributor or account avatars | Hover only |
| Multi Step Loader | First sync, long batch jobs | Real progress only, never fake steps |
| Meteors | Empty state illustration | One container, low density |
| 3D Card Effect | Marketing only | **Not** on video cards. See below |

### Explicitly not used in the application

| Component | Why |
| --- | --- |
| 3D Card Effect on video cards | 500 cards with perspective transforms destroys scroll performance and adds nothing |
| Following Pointer | Fights precise selection, which is the core dashboard interaction |
| Macbook Scroll, Hero Parallax, Container Scroll, Google Gemini Effect | Scroll hijacking. Landing page only, and only one per page |
| Background Boxes | Heavy hover grid, wrong for a data dense screen |
| SVG Mask Effect, Text Reveal Card | Hides content behind a hover. Inaccessible and slow to use |
| Card Stack | Obscures data. We have a grid for a reason |

## Rules for motion

1. **Motion must mean something in the app.** State change, progress, arrival, departure. Decoration belongs on
   the landing page.
2. **One animated background per page.** Ever.
3. **Never animate the critical path.** A user waiting to click something does not enjoy your reveal.
4. **`prefers-reduced-motion` is honoured everywhere**, not as an afterthought. Animated backgrounds become
   static gradients, reveals become instant. This is asserted in the E2E suite.
5. **Nothing important is only visible after an animation.** Content is in the DOM and readable regardless.
6. **Effects pause when off screen** and when the tab is hidden. A gradient animating in a background tab is
   just a battery drain.
7. **Duration scale:** 100 ms for a hover or a state flip, 200 ms for a panel or a sheet, 400 ms for a page
   transition. Anything longer needs a reason.
8. **Transform and opacity only** for anything that runs per frame. No animating layout properties.

## Where motion earns its place in the product

| Moment | Treatment |
| --- | --- |
| Landing hero | Animated gradient or beams, headline reveal, one spotlit call to action |
| Login and register | Quiet gradient background, no reveals on the form itself |
| Onboarding, connect account | A gentle progress feel, since OAuth round trips are otherwise jarring |
| First sync | Multi step loader with **real** stages: authorising, fetching page N, reconciling |
| Empty dashboard | An illustrated empty state with a low density effect, plus a clear next step |
| Selection bar appearing | 150 ms slide and fade. It must feel attached to the action |
| Job progress | A real progress bar, plus a subtle pulse while active. No fake indeterminate spinners |
| Job complete | A brief success flourish, once, then it settles |
| Automation canvas | Static dot grid, animated edges only while a run is executing |
| Live flow run | The active node glows, and the executed edge animates. This is information, not decoration |

## Budgets

Effects are not free. These are enforced at the Phase 1 and Phase 6 gates.

| Budget | Limit |
| --- | --- |
| Landing page JavaScript | Under 150 KB gzipped |
| Dashboard route JavaScript | Under 250 KB gzipped |
| Animated background CPU | Under 5 percent on a mid range laptop while idle |
| Scroll performance | 60 fps with 500 cards loaded. Measured, not assumed |
| Largest Contentful Paint | Under 2.0 s on the landing page, under 1.5 s on the dashboard |
| Effects per view | One animated background, at most two accent effects |

If a component cannot meet the budget, it does not ship, however good it looks.

## Accessibility, non negotiable

- `prefers-reduced-motion: reduce` disables every non essential animation. Tested.
- Text over an animated background must still meet contrast at every frame of the animation. Test the worst
  frame, not the first one.
- No effect may trap focus, swallow a keyboard event or depend on hover to reveal information.
- Nothing flashes more than three times per second.
- Every vendored component is audited for roles and labels before it is used. Ported showcase components are
  often built for looks, not for screen readers, and fixing that is our job once we copy it in.

## Theming

- Dark mode is the default. This is a tool for people who work at night, and the content is video thumbnails
  which sit better on a dark surface.
- Light mode is fully supported, not an afterthought. Every effect is checked in both.
- One accent colour, from tokens. Category and tag colours come from a fixed, contrast checked palette so a
  user cannot pick something illegible.
- All colour is a token. No hex value in a component, ever.
