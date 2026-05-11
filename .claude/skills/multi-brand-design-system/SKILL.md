---
name: multi-brand-design-system
description: How to build one design system that powers multiple sibling products with distinct brand identities — token layering (shared + brand-specific), dark glassmorphic patterns done accessibly, and motion as brand differentiation.
---

## When to use this skill

Use this skill when:

- You are starting a new product that will join an existing fleet of sibling apps sharing infrastructure
- You are designing or refactoring the token system for a multi-brand product family
- You are deciding how much to share vs. fork between sibling products
- You are building dark, glass-styled UI and want it to remain accessible and performant
- You are trying to make each sibling product feel distinct without forking the codebase

Do not use this skill for single-product design systems. The token layering described here is overkill for a single brand.

## Core idea: one system, brand-skinnable

The goal is a system where switching a single root attribute (`data-brand="a"`, `data-brand="b"`, `data-brand="c"`) re-skins the entire app to a different brand identity — without any component code changing. Components consume *semantic* tokens. Semantic tokens resolve to *brand* tokens. Brand tokens resolve to *primitive* tokens.

If you find yourself writing `if (brand === 'a') { ... }` inside a component, the system has failed and you should fix the tokens, not the component.

## Token layer architecture

Three layers. Each layer has a single job. Components only ever consume Layer 3.

### Layer 1 — Primitive tokens

Raw values. Fleet-wide. No semantics, no brand.

```css
:root {
  /* Neutrals — the dark glassmorphic base */
  --neutral-50: oklch(98% 0.005 270);
  --neutral-100: oklch(94% 0.008 270);
  --neutral-400: oklch(65% 0.015 270);
  --neutral-600: oklch(42% 0.018 270);
  --neutral-800: oklch(22% 0.020 270);
  --neutral-900: oklch(14% 0.018 270);
  --neutral-950: oklch(9% 0.015 270);

  /* Spacing scale (4px base) */
  --space-1: 0.25rem;
  --space-2: 0.5rem;
  --space-3: 0.75rem;
  --space-4: 1rem;
  --space-6: 1.5rem;
  --space-8: 2rem;
  --space-12: 3rem;
  --space-16: 4rem;

  /* Radius scale */
  --radius-sm: 0.375rem;
  --radius-md: 0.625rem;
  --radius-lg: 1rem;
  --radius-xl: 1.5rem;
  --radius-pill: 999px;

  /* Type scale (fluid) */
  --text-xs: clamp(0.75rem, 0.72rem + 0.1vw, 0.8125rem);
  --text-sm: clamp(0.875rem, 0.84rem + 0.15vw, 0.9375rem);
  --text-base: clamp(1rem, 0.95rem + 0.2vw, 1.0625rem);
  --text-lg: clamp(1.125rem, 1.05rem + 0.3vw, 1.25rem);
  --text-2xl: clamp(1.5rem, 1.3rem + 1vw, 2rem);
  --text-4xl: clamp(2rem, 1.5rem + 2.5vw, 3.25rem);
  --text-display: clamp(2.5rem, 1.5rem + 5vw, 5rem);

  /* Motion primitives */
  --duration-instant: 80ms;
  --duration-fast: 150ms;
  --duration-base: 240ms;
  --duration-slow: 400ms;
  --duration-deliberate: 640ms;

  --ease-out-quart: cubic-bezier(0.25, 1, 0.5, 1);
  --ease-out-expo: cubic-bezier(0.16, 1, 0.3, 1);
  --ease-spring: cubic-bezier(0.34, 1.56, 0.64, 1);
  --ease-sinusoidal: cubic-bezier(0.45, 0.05, 0.55, 0.95);
}
```

Primitive tokens never appear in component code. If a primitive needs to change (e.g., a neutral shifts cooler), it changes in one place and every brand inherits the change.

### Layer 2 — Brand tokens

Per-product overrides. Set at the root of each product (or via a `data-brand` attribute on `:root` in a single-codebase multi-brand setup).

```css
/* Product A — decompression */
:root[data-brand="a"] {
  --brand-hue: 268;
  --brand-accent: oklch(68% 0.22 var(--brand-hue));
  --brand-accent-soft: oklch(68% 0.22 var(--brand-hue) / 0.16);
  --brand-glow: oklch(68% 0.22 var(--brand-hue) / 0.35);
  --brand-ambient-tint: oklch(22% 0.08 var(--brand-hue));
  --brand-motion-personality: "settling";
  --brand-motion-multiplier: 1.4;
  --brand-motion-ease: var(--ease-out-expo);
}

/* Product B — guidance */
:root[data-brand="b"] {
  --brand-hue: 205;
  --brand-accent: oklch(70% 0.18 var(--brand-hue));
  --brand-accent-soft: oklch(70% 0.18 var(--brand-hue) / 0.14);
  --brand-glow: oklch(70% 0.18 var(--brand-hue) / 0.30);
  --brand-ambient-tint: oklch(20% 0.06 var(--brand-hue));
  --brand-motion-personality: "decisive";
  --brand-motion-multiplier: 0.85;
  --brand-motion-ease: var(--ease-out-quart);
}

/* Product C — mindfulness */
:root[data-brand="c"] {
  --brand-hue: 165;
  --brand-accent: oklch(72% 0.15 var(--brand-hue));
  --brand-accent-soft: oklch(72% 0.15 var(--brand-hue) / 0.12);
  --brand-glow: oklch(72% 0.15 var(--brand-hue) / 0.28);
  --brand-ambient-tint: oklch(20% 0.05 var(--brand-hue));
  --brand-motion-personality: "breath";
  --brand-motion-multiplier: 1.6;
  --brand-motion-ease: var(--ease-sinusoidal);
}
```

Brand tokens never appear in component code either. They are an implementation detail of the semantic layer.

### Layer 3 — Semantic tokens

What components consume. Intent-named.

```css
:root {
  /* Surfaces */
  --surface-canvas: var(--neutral-950);
  --surface-1: var(--neutral-900);
  --surface-2: var(--neutral-800);
  --surface-glass: oklch(100% 0 0 / 0.04);
  --surface-glass-strong: oklch(100% 0 0 / 0.08);

  /* Text */
  --text-primary: var(--neutral-50);
  --text-secondary: var(--neutral-100);
  --text-muted: var(--neutral-400);
  --text-disabled: var(--neutral-600);

  /* Borders and edges */
  --border-subtle: oklch(100% 0 0 / 0.06);
  --border-strong: oklch(100% 0 0 / 0.12);
  --edge-light: oklch(100% 0 0 / 0.08);

  /* Brand-driven semantics */
  --cta-bg: var(--brand-accent);
  --cta-fg: var(--neutral-950);
  --cta-hover-glow: var(--brand-glow);
  --focus-ring: var(--brand-accent);
  --link: var(--brand-accent);
  --ambient-tint: var(--brand-ambient-tint);

  /* Motion */
  --transition-hover: var(--duration-fast) var(--brand-motion-ease);
  --transition-reveal: calc(var(--duration-base) * var(--brand-motion-multiplier))
                       var(--brand-motion-ease);
  --transition-page: calc(var(--duration-slow) * var(--brand-motion-multiplier))
                     var(--brand-motion-ease);
}
```

Components like `Button`, `Card`, `Input`, `Tab` only reference Layer 3. This is the contract.

## Color strategy for sibling brands on a shared dark glassmorphic base

The dark glassmorphic base is fleet-shared. Brands differentiate through hue, not value.

Guidelines:

- **One brand hue per product.** Resist secondary accent colors. They dilute brand recognition and explode the contrast surface area you must audit. If you need additional accents for charts or status, derive them deterministically from the brand hue (rotate hue by 60°/120°/180°), not from a parallel palette.
- **Match chroma across the fleet.** If one brand uses an accent with chroma 0.22 and another uses 0.08, the second product will feel washed out compared to the first. Pick a chroma target (e.g., 0.18-0.22 in OKLCH) and hit it across all brands.
- **Tune lightness for the dark base.** Accents that look great on white look chalky on a dark glass surface. Push lightness to ~65-72% for visibility against deep neutrals.
- **Reserve the accent for moments, not surfaces.** Large accent-colored surfaces fight the glass. Use the accent for: primary CTA, focus ring, active state, brand glow behind the hero, single-letter brand mark. Do not paint headers or sidebars in the accent.
- **Ambient tint is your secret weapon.** Tint the page-level ambient background with a *very* desaturated, *very* dark version of the brand hue (see `--brand-ambient-tint`). This is what makes sibling products feel distinct even when no accent is on screen.

## Glassmorphic do's and don'ts

### Do

- Use blur (`backdrop-filter: blur(16px-24px)`) on small, elevated surfaces: dialogs, popovers, command palettes, floating nav.
- Add an inset 1px highlight on the top edge of glass surfaces: `box-shadow: inset 0 1px 0 var(--edge-light)`. This is the single most important glass detail.
- Tint the glass with the surface color underneath, not white. White-tinted glass looks like frosted plastic. Use `background: var(--surface-glass)` where `--surface-glass` is a low-alpha white over a tinted background — the tint comes from the ambient layer behind, not from the glass.
- Layer the ambient backdrop. A radial gradient using `--brand-glow` placed off-center, a faint noise texture, and the canvas color stacked together make glass feel alive.
- Provide a solid fallback for browsers where you disable `backdrop-filter`:

```css
.glass-surface {
  background: var(--surface-glass);
  backdrop-filter: blur(20px) saturate(140%);
  -webkit-backdrop-filter: blur(20px) saturate(140%);
  box-shadow:
    inset 0 1px 0 var(--edge-light),
    0 20px 40px -20px oklch(0% 0 0 / 0.5);
}

@supports not (backdrop-filter: blur(1px)) {
  .glass-surface {
    background: var(--surface-2);
  }
}
```

### Don't

- Don't blur full-bleed sticky headers on long-scroll pages. Performance dies on mid-tier devices. Use a tinted opaque background with a 1px bottom border instead.
- Don't stack more than two glass layers. Glass-over-glass-over-glass becomes muddy and contrast is no longer computable.
- Don't use heavy drop shadows under glass. The shadow should be soft, large, and far below the surface (`0 24px 64px -24px oklch(0% 0 0 / 0.5)`), not the default Material 4dp shadow.
- Don't put dense body text on glass. Use glass for cards, dialogs, and CTAs. Use opaque surfaces for long-form reading.
- Don't forget the contrast audit. Glass changes effective background. Always test text contrast against the brightest pixel that can appear underneath the surface.

## Typography pairing across brands

Share typography across the fleet. One or two families, well-paired, is plenty.

- **Display family.** A confident sans (Inter Display, General Sans, or similar). Used for hero headings and brand marks. Shared across all brands.
- **Body family.** A workhorse sans optimized for screen reading (Inter, Söhne, IBM Plex Sans, similar). Shared.
- **Optional mono.** For credentials, code, timestamps, callout numerals. Shared.

Differentiation comes from *use*, not family:

- One brand may favor tight tracking, large optical sizes, and a heavy weight in display.
- Another may favor airy tracking, light weights, and uppercase eyebrows.
- A third may use lowercase exclusively in display for a softer voice.

These are token-level decisions you can encode:

```css
:root[data-brand="a"] {
  --display-tracking: -0.02em;
  --display-weight: 600;
  --display-case: none;
}

:root[data-brand="b"] {
  --display-tracking: -0.01em;
  --display-weight: 700;
  --display-case: none;
}

:root[data-brand="c"] {
  --display-tracking: 0;
  --display-weight: 400;
  --display-case: lowercase;
}
```

## Motion personality per brand

Motion is the most underrated brand lever. Sibling products with identical color and type will still feel completely different if their motion personalities diverge.

Define motion personality at the brand token layer using a multiplier and an easing preference:

```css
:root[data-brand="a"] {
  --brand-motion-multiplier: 1.4;   /* slower than baseline */
  --brand-motion-ease: var(--ease-out-expo);
}

:root[data-brand="b"] {
  --brand-motion-multiplier: 0.85;  /* faster than baseline */
  --brand-motion-ease: var(--ease-out-quart);
}

:root[data-brand="c"] {
  --brand-motion-multiplier: 1.6;   /* slowest, breath-paced */
  --brand-motion-ease: var(--ease-sinusoidal);
}
```

Personality descriptors to choose between:

- **Settling** — long durations, deep ease-out. Things arrive and rest. Good for decompression, journaling, meditation.
- **Decisive** — short durations, sharp ease-out with optional slight overshoot. Things arrive with intent. Good for productivity, finance, dashboards.
- **Breath** — very long durations on ambient elements (background pulse, subtle scale), crisp on interaction. Sinusoidal easing. Good for wellness, mindfulness.
- **Kinetic** — medium durations, springy easing with overshoot. Good for creative tools, social, entertainment.
- **Mechanical** — short, linear or near-linear, exact. Good for technical and developer tools.

Pick one personality per brand. Document it. Do not mix.

## Component variants vs. forks

Default to variants. Fork only when the divergence is total.

- **Variants.** A `Button` has `primary`, `secondary`, `ghost`, `destructive` variants. All brands share all variants. The variant resolves to semantic tokens which resolve to brand tokens.
- **Forks.** A `HeroSection` for the decompression product may have nothing in common with the guidance product's hero. Fork it. Do not try to build one component that handles both via 18 props.

Test for fork-worthiness: if more than 30% of the component's markup or behavior differs between brands, fork. If less, parameterize.

Avoid the middle ground where a single component accepts a `brand` prop and contains conditional logic. That is the worst of both worlds.

## Tailwind-style example: brand-aware button

```tsx
// Button.tsx — never references brand directly
export function Button({ variant = "primary", children, ...props }) {
  return (
    <button
      className={cn(
        "inline-flex items-center justify-center",
        "rounded-md px-4 py-2 text-sm font-medium",
        "transition-[background,box-shadow,transform]",
        "duration-[var(--duration-fast)] ease-[var(--brand-motion-ease)]",
        "focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-[var(--focus-ring)] focus-visible:ring-offset-2 focus-visible:ring-offset-[var(--surface-canvas)]",
        "motion-reduce:transition-none",
        variant === "primary" && [
          "bg-[var(--cta-bg)] text-[var(--cta-fg)]",
          "hover:shadow-[0_0_32px_var(--cta-hover-glow)]",
          "active:translate-y-px",
        ],
        variant === "secondary" && [
          "bg-[var(--surface-glass)] text-[var(--text-primary)]",
          "shadow-[inset_0_1px_0_var(--edge-light)]",
          "hover:bg-[var(--surface-glass-strong)]",
        ],
        variant === "ghost" && [
          "bg-transparent text-[var(--text-secondary)]",
          "hover:bg-[var(--surface-glass)]",
        ],
      )}
      {...props}
    >
      {children}
    </button>
  );
}
```

Switching `<html data-brand="a">` to `<html data-brand="b">` re-skins every button in the app. No component code changes.

## Accessibility

Treat these as hard requirements enforced at the token layer.

### Contrast

- Body text on any surface: ≥ 4.5:1.
- Large text (≥ 18pt or 14pt bold) and UI controls: ≥ 3:1.
- Disabled text: ≥ 3:1 (do not hide affordance via contrast).
- Test contrast against the *effective* background. On glass surfaces, that means the brightest pixel under the surface, not the glass tint.

### Focus

Every interactive element must have a visible focus state. Use `--focus-ring` (the brand accent) at full saturation with a ring offset against the canvas:

```css
*:focus-visible {
  outline: none;
  box-shadow:
    0 0 0 2px var(--surface-canvas),
    0 0 0 4px var(--focus-ring);
}
```

Never remove focus rings to "clean up" a design. Make them beautiful instead.

### Reduced motion

Encode `prefers-reduced-motion` at the token layer so every component inherits it automatically:

```css
@media (prefers-reduced-motion: reduce) {
  :root {
    --duration-instant: 1ms;
    --duration-fast: 1ms;
    --duration-base: 1ms;
    --duration-slow: 1ms;
    --duration-deliberate: 1ms;
    --brand-motion-multiplier: 0;
  }
}
```

For ambient looping animations (e.g., the breath pulse in a mindfulness brand), guard them explicitly:

```css
@media (prefers-reduced-motion: no-preference) {
  .ambient-pulse {
    animation: pulse 6s var(--ease-sinusoidal) infinite;
  }
}
```

### Hit targets

Interactive elements must have at least 44x44px hit area. Use padding to expand hit area when the visual element is smaller.

### Color is never the only signal

Errors get an icon and a label, not just red. Active tabs get weight or position change, not just color. Sortable column indicators get a directional glyph, not just a hue shift.

## Quick checklist before shipping a screen

- [ ] No raw color literals, no pixel literals, no inline durations
- [ ] Every interactive element has hover, focus, active, disabled defined
- [ ] Focus ring is visible against every surface it can appear over
- [ ] Reduced-motion handling is in place (not deferred)
- [ ] Glass surfaces have inset edge light and a solid fallback
- [ ] No more than two stacked glass layers
- [ ] Text on glass has been contrast-tested against worst-case background
- [ ] Switching `data-brand` re-skins the screen without code changes
- [ ] Motion personality matches the product, not the developer's defaults
- [ ] The screen would not be mistaken for a stock AI-template hero
