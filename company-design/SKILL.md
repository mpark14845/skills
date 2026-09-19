---
name: company-design
description: Defines the company brand design system — color tokens, typography, spacing, motion, and core UI components (buttons, cards, pills, nav) — for a confident, plain-spoken, crimson-accented brand. Use this skill whenever building or reviewing UI for this company's products or marketing pages, so colors, type, spacing, and component patterns (e.g. the offset-shadow card motif) match the established brand rather than being invented per page. Pairs with `vanilla-js-frontend` for implementation.
---

## 1. Brand Personality

Confident, plain-spoken, and warm — a local AI consultancy, not a faceless SaaS.
Design choices should feel **direct and human**: bold headlines, generous
whitespace, no jargon in the copy, no clutter in the layout.

- **Primary mood**: clean light-mode product UI, punctuated by one saturated
  crimson accent and occasional full dark sections for contrast.
- **One color dominates**: crimson is used sparingly but decisively (CTAs,
  icons, key words in headlines) — never as a background wash.
- **Motif**: a soft white card, offset behind a pale rose shadow shape,
  slightly larger and shifted down-right. Reuse this on any card that should
  feel "featured" (pricing, testimonials, advantage lists).

---

## 2. Color Tokens

```css
:root {
  /* Brand */
  --nu-brand-600:   #F94F51;  /* primary crimson — buttons, links, icon accents */
  --nu-brand-100:   #FEF2F2;  /* pale rose — pill backgrounds, icon circle fills */
  --nu-accent-600:  #F9642E;  /* orange — headline highlight word, secondary accent */

  /* Ink / neutrals */
  --nu-ink-900:     #0F172A;  /* headline text, dark section background */
  --nu-ink-700:     #1E293B;  /* card fill / chip fill on dark sections */
  --nu-ink-600:     #334155;  /* borders on dark sections */
  --nu-body-600:    #475569;  /* body copy on light backgrounds */
  --nu-muted-400:   #8B98AA;  /* muted/secondary text on dark backgrounds */
  --nu-line-200:    #E2E8F0;  /* hairline borders, card outlines on light bg */
  --nu-caption-400: #9AA5B1; /* footer text, captions, page numbers */

  /* Surfaces */
  --nu-surface:      #FFFFFF; /* cards, modals, nav bar */
  --nu-bg:           #F8FAFC; /* page background (light mode) */
  --nu-bg-dark:      #0F172A; /* page background (dark sections/mode) */
  --nu-surface-dark: #1E293B; /* card fill on dark sections */
}
```

**Usage rules**

- Backgrounds are either `--nu-bg` (near-white, cool — never cream/beige) or
  `--nu-bg-dark` (deep navy). Alternate them section-to-section for a
  light/dark "sandwich" rhythm on long pages; don't mix a third background
  color in.
- `--nu-brand-600` is the only saturated color allowed on buttons, active
  states, and links. `--nu-accent-600` (orange) is reserved for highlighting
  one word in a headline or a secondary stat — never both colors in the same
  small element.
- Text on light backgrounds: `--nu-ink-900` for headlines, `--nu-body-600`
  for paragraphs. Text on dark backgrounds: `#FFFFFF` for headlines,
  `--nu-muted-400` for paragraphs. Never place `--nu-body-600` on a dark
  background or `--nu-muted-400` on a light one — both fail contrast.

---

## 3. Typography

```css
:root {
  --nu-font-head: "Inter", "Arial", system-ui, sans-serif;   /* headlines — bold, tight */
  --nu-font-body: "Inter", "Calibri", system-ui, sans-serif; /* body copy */
}
```

The live site uses a bold grotesk (Inter-like) for everything. If exact-match
fonts aren't available, any bold system sans-serif is an acceptable
substitute — **do not switch to a serif or a rounded display font.**

| Element | Size | Weight | Color |
|---|---|---|---|
| Hero headline | 40–56px | 800 (extrabold) | `--nu-ink-900`, highlight word in `--nu-accent-600` |
| Section headline | 28–32px | 800 | `--nu-ink-900`, highlight word in `--nu-brand-600` |
| Card / component title | 16–18px | 700 | `--nu-ink-900` |
| Body / paragraph | 14–16px | 400 | `--nu-body-600` |
| Button label | 13–14px | 700 | white (filled) / `--nu-ink-900` (outline) |
| Caption / footer / eyebrow | 10–12px | 600–700, uppercase for eyebrows | `--nu-caption-400` or `--nu-brand-600` |

Headlines are tight (`line-height: 1.05–1.1`), body copy is relaxed
(`line-height: 1.4–1.5`).

---

## 4. Spacing & Radius

4px grid, same discipline as the rest of the product suite:

```css
:root {
  --nu-space-1: 4px;
  --nu-space-2: 8px;
  --nu-space-3: 12px;
  --nu-space-4: 16px;
  --nu-space-6: 24px;
  --nu-space-8: 32px;
  --nu-space-12: 48px;
  --nu-space-16: 64px;

  --nu-radius-sm: 6px;    /* pills, chips, small buttons */
  --nu-radius-md: 12px;   /* buttons, inputs */
  --nu-radius-lg: 16px;   /* cards, panels, image frames */
  --nu-radius-full: 9999px; /* logo mark, avatar, icon circles */
}
```

Section vertical padding: `--nu-space-16` (desktop) / `--nu-space-8` (mobile).
Card internal padding: `--nu-space-6`. Minimum gap between adjacent cards or
content blocks: `--nu-space-6`.

---

## 5. Motion

```css
:root {
  --nu-duration-sm: 150ms;
  --nu-duration-md: 250ms;
  --nu-ease: cubic-bezier(0.4, 0, 0.2, 1);
}
```

Use for hover/active/focus state changes only — background tint shifts,
subtle lift (`translateY(-2px)`) on interactive cards, opacity fades on
modals. Avoid bouncy or springy easing; the brand is calm and direct.

---

## 6. Core Components

### Buttons

- **Primary**: `--nu-brand-600` fill, white bold text, `--nu-radius-md`,
  no border. Hover: darken fill ~8%. Used for the one clear action per
  section ("Book a Consultation", "Start Your AI Journey").
- **Secondary**: white (or transparent on dark sections) fill, 1px
  `--nu-line-200` border, `--nu-ink-900` (or white on dark) text. Same
  radius and padding as primary so they sit evenly side-by-side.
- Always pair at most one primary + one secondary button together — never
  two primaries.

### Pills / Eyebrow badges

Small rounded-full or `--nu-radius-sm` chip, `--nu-brand-100` fill,
`--nu-brand-600` bold uppercase text, used above a headline to label the
section ("AI CONSULTING FOR THE SOUTHERN TIER"). On dark backgrounds, swap
the fill for `--nu-ink-700`.

### Cards — "offset shadow" motif

The signature brand pattern. A white (or `--nu-ink-700` on dark) rounded
card sits slightly up-left of a solid `--nu-brand-100` shape of the same
size, peeking out bottom-right:

```css
.card-offset {
  position: relative;
}
.card-offset::before {
  content: "";
  position: absolute;
  inset: 8px -8px -8px 8px; /* shifted down-right */
  background: var(--nu-brand-100);
  border-radius: var(--nu-radius-lg);
  z-index: -1;
}
.card-offset {
  background: var(--nu-surface);
  border: 1px solid var(--nu-line-200);
  border-radius: var(--nu-radius-lg);
  box-shadow: 0 8px 24px rgba(15, 23, 42, 0.08);
}
```

Reserve this for one "hero" card per page (an advantage list, a pricing
card, a featured testimonial) — it loses impact if every card uses it.

### Plain cards

Standard content cards (three-up feature grids, stat callouts) skip the
offset shadow: white fill, `--nu-line-200` 1px border, `--nu-radius-lg`,
soft `box-shadow: 0 2px 8px rgba(15,23,42,0.06)`.

### Checklist items

Icon: `--nu-radius-full` circle, `--nu-brand-100` fill, 1.5px `--nu-brand-600`
stroke, checkmark glyph in `--nu-brand-600`. Label text in `--nu-ink-900`,
14px. Stack vertically with `--nu-space-4` between items — never wrap into a
grid.

### Icon circles (feature/process steps)

`--nu-radius-full` circle, `--nu-brand-100` fill (or `--nu-ink-700` on dark),
icon or numeral in `--nu-brand-600`, sized ~48–60px. Used at the top of
feature cards and process-step cards.

### Navigation bar

White background, logo mark + wordmark on the left, text links centered/left
of a single primary button on the right, bottom hairline `--nu-line-200`
border, no shadow.

### Logo mark

`--nu-radius-full` circle, `--nu-brand-600` fill, single bold white letter
centered, paired with the bold wordmark in `--nu-ink-900` (or white on dark
backgrounds).

---

## 7. Layout Patterns

- **Two-column hero**: headline + copy + buttons on the left (~55% width),
  a featured offset-shadow card or product screenshot on the right.
- **Light/dark sandwich**: alternate `--nu-bg` and `--nu-bg-dark` sections
  down the page for rhythm; keep hero and closing CTA sections dark or
  light deliberately (dark reads more "premium/serious", light reads more
  "everyday/approachable").
- **Three-up feature grid**: equal-width plain cards, icon circle + title +
  2-line description, on `--nu-bg`.
- **Stat row**: 3–4 equal cards, large `--nu-brand-600` numeral (32–40px
  bold) over a short muted label.
- Max content width: **1200px**, centered, with at least `--nu-space-6` side
  padding on mobile.

---

## 8. Do / Don't

**Do**
- Keep one saturated accent color per element — crimson OR orange, never
  both.
- Left-align body copy and lists; center only hero headlines and pill
  badges.
- Use the offset-shadow card once per page, on the most important content.
- Keep dark sections truly dark (`--nu-bg-dark`) with white/muted text —
  never dark text on a dark background.

**Don't**
- Don't default to cream/beige backgrounds — use `--nu-bg` (cool near-white)
  or `--nu-bg-dark`.
- Don't add decorative color bars, accent stripes, or underlines beneath
  headlines — the brand carries weight through bold type and whitespace,
  not lines.
- Don't mix serif type in — headlines and body are both sans-serif.
- Don't stack more than one primary button in the same view.
- Don't use the crimson accent as a large background fill — it's a small,
  high-contrast accent, not a section color.
