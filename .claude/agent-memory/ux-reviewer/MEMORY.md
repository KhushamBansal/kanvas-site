# Kanvas UX Reviewer — Persistent Memory

## Design Token Summary
- Primary: `#00b39f` (teal-green) via `$primary`
- Secondary: `#00d3a9` via `$secondary`
- Dark bg: `#121212` via `$dark`; body-bg `#010101`
- Muted text: `$casper: #b1b6b8`
- White: `$white`
- Font: "Qanelas Soft" → Space Grotesk → Inter fallback
- Spacer unit: Bootstrap-based `$spacer` (1rem)

## Two Parallel Glass Systems (Known Issue)
The codebase has TWO glass systems that must stay in sync:
1. `_hero-glass.scss` — `.glass`, `.glass--dark`, `.glass--thin`, `.button` (hero page variant, blur 32px, border-radius 28px)
2. `_section-transitions.scss` — `.glass-card` (site-wide utility class, blur 20px, radius 16px)
3. Individual component glass (cap-card, community-card, persona-card, demo-container) — blur 16-20px, radius 16px
All use `backdrop-filter: blur() saturate(180%)` + `inset 0 1px 0` top highlight pattern.

## Animation Conventions (Confirmed)
- Primary easing: `cubic-bezier(0.16, 1, 0.3, 1)` (expo-out feel) — used in all new hover transitions
- Hover lift: `translateY(-4px)` on cards, `translateY(-3px)` on buttons
- Active press: `translateY(-1px)` with `transition-duration: 0.1s`
- Scroll reveal duration: 0.7s–1.2s with `power3.out` in GSAP
- Stagger interval: 0.1s between grid children
- Card hover transition: 0.4s
- `prefers-reduced-motion`: handled in `_hero-glass.scss` (global `*` selector) and `_section-transitions.scss` (scoped to `.section-reveal`)

## GSAP Architecture (main.js)
- GSAP + ScrollTrigger loaded externally (footer.html) — CRITICAL: ScrollTrigger CDN loads BEFORE gsap.min.js in footer (line 75 before 76). Plugin registered before core = crash risk on cold cache.
- `initSectionReveals()` and `initSectionDividers()` have been REMOVED. Replaced by `initScrollAnimations()` (scrub-based system).
- `animateCounters()` uses `requestAnimationFrame` manually (not GSAP), fired via ScrollTrigger `once:true` — correct pattern.
- `initMarquee()` duplicates innerHTML for infinite scroll effect.
- Header timeline (tl) still runs at top-level on script parse — NOT inside DOMContentLoaded. Risk if DOM not ready.
- `initScrollAnimations()` now calls `document.body.prepend(ambient)` — but `.hero-glass.scss` already has `.ambient` / `.orb` fixed background system. Two independent fixed overlay systems now exist simultaneously.

## Scroll System — New Scrub Implementation (Reviewed March 2026)
- `initScrollAnimations()` in main.js: full scrub-based parallax system
- Section order: hero-glass (#hero) → customers → hero-section → demo → capabilities → community → browser
- Known selector bugs: `.capabilities-subtitle` queried inside `.community-header` (wrong partial class reuse in community.html)
- Known structural bug: `.capabilities-grid` is INSIDE `.capabilities-header` div — trigger set to `.capabilities-section` but end to `center 50%` of section which may resolve correctly; grid selector valid.
- Hero recession: `#hero` (glass card) scrubbed scale(0.92) opacity(0.4) y(-40) — conflicts with `hero-glass.js` which writes inline `--tilt-x/--tilt-y` and `perspective(1200px) rotateX/rotateY` transforms on same element. GSAP will override or fight the inline style transform.
- Ambient orb system: new `.scroll-orb` elements use `filter: blur(100px)` (heavy) + `will-change: transform, opacity`. Original `.orb` system uses `filter: blur(18px)`. Both run simultaneously as fixed layers — GPU overdraw risk.
- Browser section: `.browser-stand` ScaleX from 0.5 has no `transform-origin: center` set in CSS (defaults to `50% 50%`, which is correct — acceptable).
- `prefers-reduced-motion` guard correctly gates the entire `initScrollAnimations()` function with early return.
- Section dividers: dynamically injected after each of 5 named sections — guard against double-injection exists (`nextElementSibling` check).

## Known Anti-Patterns / Issues to Watch
- SCRIPT LOAD ORDER BUG (footer.html line 75-76): ScrollTrigger CDN loads before gsap.min.js — plugin registered before core is available. Must swap order.
- Header nav animation: `rotation: 360` on `.btn-secondary` is a jarring full-spin on page load — against the vision (still present, tl runs at parse time)
- Color literals scattered: `#00b399`, `#00b39f`, `#00B39F` all used for the same primary color — should use `$primary` token
- `transition: 0.3s` bare shorthand used in `.community-primary` and social links — missing easing function
- `.glass-card` blur (20px) vs component cards (16px) — slight inconsistency but acceptable
- `scroll-behavior: smooth` correctly moved into `@media (prefers-reduced-motion: no-preference)` block in _section-transitions.scss — RESOLVED
- Two fixed ambient overlay systems: old `.ambient/.orb` (blur 18px, CSS float animation) + new `.scroll-ambient/.scroll-orb` (blur 100px, GSAP scrub) — GPU overdraw on every scroll
- GSAP scrub recession on `#hero` conflicts with `hero-glass.js` writing inline transform via CSS custom properties on same element
- Wrong class selector in community section: `comHeader.querySelector('.capabilities-subtitle')` — copies class from capabilities partial instead of community-specific class
- `.scroll-orb` filter:blur(100px) on three fixed elements is expensive — prefer lower blur or SVG filter approach
- `stagger` with `scrub` in GSAP: stagger+scrub combination causes each card to have its own independent ScrollTrigger internally — with same start/end markers this creates n×ScrollTrigger instances for each card grid

## Color Palette Deviation from Vision
The site uses teal/cyan (#00b39f) as primary, NOT the recommended deep indigo/violet palette from the design brief. This is the established Kanvas brand color — do NOT recommend changing it to indigo. The palette is intentionally teal-forward.

## Files Reference
- Variables: `/assets/scss/_variables_project.scss`
- SCSS entry: `/assets/scss/_styles_project.scss`
- Glass base: `/assets/scss/_hero-glass.scss`
- Transitions: `/assets/scss/_section-transitions.scss`
- JS: `/static/scripts/main.js`
