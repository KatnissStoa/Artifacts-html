---
name: artifact-visual
description: >-
  Mandatory visual design layer for ALL web artifacts. Applies a strict
  anti-slop design contract with OKLCH color tokens, intentional typography,
  and purposeful motion. This skill is NOT routed — it runs for every website
  regardless of artifact type (landing page, dashboard, form, interactive tool).
  Downstream skills (artifact-dashboard, artifact-form, artifact-interactive,
  fallback) inherit the visual foundation established here.
---

# Visual Design Layer (Mandatory)

> **This skill runs for every web artifact.** The project has already been
> initialized via `web-artifacts-builder` (Step 1 of the main router). This
> skill establishes the visual design foundation that ALL downstream artifact
> types must follow — including dashboards, forms, interactive tools, and
> generic landing pages.

## Design System Selection

Before configuring tokens, select a design system to guide the visual output.
Read `design-system-index.md` (in this directory) for the full brand directory.

### Selection logic

1. **Analyze the user's request** for industry, tone, and visual cues:
   - Industry keywords (VC, restaurant, pet product, SaaS, etc.)
   - Explicit style requests ("dark and premium", "warm and friendly", etc.)
   - If the content-sourcer ran: use scraped `brandColors`, `fonts`, and site
     atmosphere as additional signals

2. **Pick the best-matching brand** from the index. Consult the Brand
   Directory table first; if the user described a mood rather than an
   industry, use the Style Quick-Reference groupings at the bottom of the
   index file.

3. **Download the design system**:
   ```bash
   npx getdesign@latest add <brand-id>
   ```
   This creates a `DESIGN.md` file in the project root.

4. **If content-sourcer extracted brand signals** (`brandColors` / `fonts`
   in `business-data.json`), note them — they will be used as overrides on
   top of the downloaded DESIGN.md (e.g., replacing the template's primary
   color with the actual brand color).

### When to skip selection

Skip **only** if:
- The user explicitly says "don't use any reference design" or provides
  their own complete design spec (colors, fonts, spacing, etc.)
- You genuinely cannot find a reasonable visual match among the 66 brands

When skipped, the design contract below provides its own default values.

### Priority

- **User's explicit style request** > auto-detected industry match
- **Scraped brand signals** (from content-sourcer) are used as overrides on
  the selected DESIGN.md, not as a reason to skip selection

---

## Design Source Priority

This skill operates in one of two modes depending on whether a `DESIGN.md`
file exists in the project root (from the selection step above):

### Mode A: DESIGN.md Present (primary path)

The downloaded `DESIGN.md` is the **highest-priority** source of truth for
all visual decisions — color palette, typography, component styling, layout
philosophy, and do's/don'ts.

This skill's role in Mode A is:
- **Derive** OKLCH tokens, font choices, and motion intensity from DESIGN.md
- **Enforce** engineering quality rules that are style-independent (OKLCH
  conversion, font loading strategy, image strategy, interaction states,
  accessibility, pre-bundle validation)
- **Override** DESIGN.md only when it conflicts with a hard engineering
  constraint (e.g., a font specified in DESIGN.md is not available on
  Google Fonts — pick the closest alternative)

If `business-data.json` contains `brandColors` or `fonts` extracted by the
content-sourcer, use them as **overrides on top of DESIGN.md** — e.g.,
replace the template's primary color with the actual brand color while
keeping the overall visual atmosphere from DESIGN.md.

### Mode B: No DESIGN.md (fallback path)

When selection was skipped (no suitable design system found), this skill's
design contract becomes the **highest-priority** source of truth for all
visual decisions. Follow Steps 1–6 below using the default values.

When the user's PRD or query conflicts with a rule below (e.g., PRD says
"use Inter" but the contract forbids it), **follow the skill**. The user
defines *what* to build; this skill defines *how* it must look and behave.

If a conflict is genuinely irreconcilable, ask the user before proceeding —
never silently skip or downgrade a rule.

---

## Execution Protocol

**You MUST complete each step in order. Do not write section/component code
until Steps 1–4 are done.** Each step has a verification check — run it
before moving on.

### Step 0: Install dependencies

```bash
pnpm add framer-motion
```

### Step 1: Color Tokens (OKLCH)

Create or update `src/index.css` with OKLCH custom properties. These are the
**only** color values your section code may reference (via `var()`).

**Mode A (DESIGN.md present):** Read the "Color Palette & Roles" section of
`DESIGN.md`. Convert every hex/RGB value to OKLCH and map semantic roles
(primary, accent, background, surface, text, border, success, warning, error)
to the `--color-*` variable system below. Use the DESIGN.md's color
philosophy (warm vs cool neutrals, tinted vs pure grays, light vs dark base)
to guide the overall palette mood. If `business-data.json` has `brandColors`,
use them as the primary/accent, keeping the DESIGN.md's neutral tinting
strategy.

**Mode B (no DESIGN.md):** If the user's PRD specifies brand colors in
hex/RGB, convert them to OKLCH and slot them into the roles below. Otherwise
use the default example values.

```css
@layer base {
  :root {
    /* Primary — derived from user's brand color, converted to OKLCH */
    --color-primary: oklch(55% 0.15 240);
    --color-primary-hover: oklch(48% 0.15 240);
    --color-primary-container: oklch(92% 0.03 240);

    /* Neutrals — TINTED toward primary hue, never pure gray */
    --color-text-primary: oklch(15% 0.01 240);
    --color-text-secondary: oklch(40% 0.01 240);
    --color-bg: oklch(98% 0.005 240);
    --color-surface: oklch(96% 0.008 240);
    --color-border: oklch(88% 0.01 240);

    /* Semantic */
    --color-success: oklch(55% 0.12 145);
    --color-warning: oklch(70% 0.15 85);
    --color-error: oklch(55% 0.18 25);
    --color-info: oklch(55% 0.10 240);

    /* Accent — if PRD has a secondary brand color */
    --color-accent: oklch(65% 0.20 25);

    /* shadcn HSL bridge — map from OKLCH values for shadcn components.
       Calculate approximate HSL equivalents of the OKLCH tokens above.
       This ensures shadcn's Button, Accordion, etc. match the palette. */
    --background: 220 20% 98%;
    --foreground: 220 28% 17%;
    --primary: 220 80% 30%;
    --primary-foreground: 0 0% 100%;
    /* ... fill remaining shadcn vars from OKLCH-derived values ... */
  }
}
```

**Rules:**
- ❌ NEVER use `#000` or `#fff` — always slightly off-black/off-white OKLCH tones
- ❌ NEVER use flat gray palettes — all neutrals must be tinted toward primary hue
- ❌ NEVER keep arbitrary RGB/hex values in component code — convert to OKLCH tokens
- ❌ NEVER use purple gradients, rainbow gradients, or "AI default" gradient combos
- ❌ NEVER use color alone to indicate state — always pair with text, icon, or layout cue
- ✅ Derive dark mode by inverting OKLCH lightness (L: 0.98→0.08) + adjusting chroma
- ✅ Tinted shadows/glows toward primary hue, never flat gray shadows
- ✅ WCAG 2.2 AA minimum: 4.5:1 for normal text, 3:1 for large text

**Verification:**
```bash
# Must have at least 8 oklch tokens defined
grep -c "oklch" src/index.css
# Must be 0 — no raw hex/rgb in section code (SVGs may use var() or OKLCH)
grep -rn "#[0-9a-fA-F]\{3,8\}\|rgb(" src/sections/ || echo "PASS"
```

### Step 2: Typography

Choose fonts and configure them. Load via JavaScript (not `<link>` in
`index.html`) to avoid html-inline bundler errors with external URLs.

**Mode A (DESIGN.md present):** Read the "Typography Rules" section of
`DESIGN.md`. Use the specified font families for display and body roles. If
the DESIGN.md specifies a proprietary/unavailable font (e.g., "SF Pro"),
pick the closest Google Fonts alternative. Apply its size scale, weight
hierarchy, and line-height specs. The engineering constraints below still
apply (min 16px body, max 2 families, `clamp()` fluid scaling, banned font
list).

**Mode B (no DESIGN.md):** Use the default font strategy below.

```ts
// src/lib/fonts.ts — load Google Fonts dynamically
export function loadFonts() {
  if (document.querySelector('link[data-fonts]')) return;
  const link = document.createElement('link');
  link.rel = 'stylesheet';
  link.href = 'https://fonts.googleapis.com/css2?family=...&display=swap';
  link.setAttribute('data-fonts', '');
  document.head.appendChild(link);
}
```

Call `loadFonts()` in `src/main.tsx` before `createRoot()`.

Extend `tailwind.config.js`:
```js
fontFamily: {
  display: ['"DM Serif Display"', 'serif'],   // or another expressive face
  body: ['"DM Sans"', 'sans-serif'],           // or another readable face
}
```

**Font Strategy:**
- **Display / Headings**: ONE expressive face — e.g., "DM Serif Display", "Playfair Display", "Space Grotesk"
- **Body / UI**: ONE refined readable face — e.g., "DM Sans", "Source Sans 3", "Geist"
- ❌ NEVER default to Inter, Roboto, Arial, Open Sans, or Helvetica as primary font
- ❌ NEVER use more than 2 font families (1 display + 1 body)
- ❌ NEVER set body text below 16px

**Type Scale** (modular, ratio ~1.25×):

| Token       | Size                           | Weight  | Line-Height | Use                       |
|-------------|--------------------------------|---------|-------------|---------------------------|
| `text-hero` | clamp(2.5rem, 5vw, 3.5rem)     | 700–800 | 1.1         | Hero headlines            |
| `text-h1`   | clamp(2rem, 4vw, 2.75rem)      | 700     | 1.15        | Page titles               |
| `text-h2`   | clamp(1.5rem, 3vw, 2rem)       | 600     | 1.2         | Section titles            |
| `text-h3`   | 1.25rem                        | 600     | 1.3         | Card titles, dialog titles|
| `text-body` | 1rem (16px)                    | 400     | 1.6         | All body text             |
| `text-sm`   | 0.875rem                       | 400     | 1.5         | Helper text, captions     |
| `text-xs`   | 0.75rem                        | 500     | 1.4         | Badges, chips, overlines  |

- Use `clamp(min, preferred, max)` for fluid scaling
- Max line width: **60–65ch** for readable paragraphs
- Button labels: `font-weight: 500–600`, `letter-spacing: 0.02em`, sentence case (not ALL CAPS)

**Verification:**
```bash
grep -q "font-display\|font-body" tailwind.config.js && echo "PASS" || echo "FAIL"
grep -rn "Inter\|Roboto\|Arial\|Open Sans\|Helvetica" tailwind.config.js && echo "FAIL: banned font" || echo "PASS"
```

### Step 3: Image Strategy

**Decide before writing any section code.** Pick one approach and note your
choice — do not defer this decision.

**Mode A note:** If DESIGN.md describes an image philosophy (e.g.,
"full-bleed photography", "cinematic imagery", "illustration-driven"),
use that as guidance for which option to pick and how images are styled
(filters, aspect ratios, overlays).

| Option | When to use | How |
|--------|-------------|-----|
| **A: AI-generated images** | Product pages, lifestyle shots, hero visuals | Use imagegen skill → save as file → reference in code. For single-HTML bundle, compress and base64-encode. |
| **B: Unsplash/Pexels URLs** | When speed matters, concepts are generic | Use absolute `https://` URLs. `html-inline` only resolves relative paths — external URLs pass through intact. See **Unsplash Selection Rules** below. |
| **C: High-quality SVG illustration** | Abstract/tech products, diagrams | Must include gradients, shadows, detail layers, varied stroke weights. NOT just basic geometric primitives (circles + rectangles). |
| **D: User-provided assets** | User attaches images | Use directly. |
| **E: Scraped assets** | Content sourcer ran (Step 0) | Reference images from `src/data/assets/`. This takes priority over B when scraped images are available. |

**Unsplash Selection Rules (Option B):**

When using Unsplash/Pexels, follow these rules to avoid generic stock feel:

| Rule | Detail |
|------|--------|
| **Keyword construction** | Combine: `[industry term] + [scene/mood] + [DESIGN.md atmosphere]`. E.g., for a VC site with dark premium design → `"venture capital modern office dark"`, NOT just `"business"`. |
| **Style match** | Dark DESIGN.md → dark/moody photos. Warm DESIGN.md → warm-lit lifestyle. Monochrome DESIGN.md → B&W or desaturated images. |
| **Sizing by section** | Hero: `w=1400&h=800&fit=crop` (16:9). Team/avatar: `w=600&h=800&fit=crop` (3:4). Portfolio logo: `w=400&h=400&fit=crop` (1:1). About: `w=1200&h=600&fit=crop`. |
| **No reuse** | Every section must use a different image. Never repeat the same photo across sections. |
| **No generic** | Ban: "happy team in office", "handshake", "laptop on desk". Prefer: industry-specific, editorial-quality, with clear subject and intentional composition. |

**Priority when content sourcer was used (Step 0 ran):**
If `src/data/assets/` contains downloaded images, you **must** use them
(option E) as the primary image source. Fall back to option B only for
sections where no relevant scraped image exists. Do NOT default to Unsplash
when scraped images are available — that defeats the purpose of scraping.

**Image Classification Mapping (CRITICAL when using scraped images):**

When `business-data.json` images contain `imageType` and `associatedName`
fields, you MUST use these for image assignment — NOT sequential index
guessing (photo-0, photo-1, photo-2...). The scraper classifies images by
their page context:

| Image mapping rule | Details |
|---|---|
| **Hero background** | Use images with `imageType: 'scene'` or `imageType: 'banner'` that are large (width > 800). **NEVER** use `imageType: 'og-image'` as a hero background — OG images are typically small logos/icons that look terrible at full-bleed scale. |
| **Team member photos** | Use images with `imageType: 'headshot'`. Match `associatedName` to the team member's name. If `team[]` entries in business-data.json have `imageUrl`, use those directly. |
| **Portfolio / company logos** | Use images with `imageType: 'product-logo'` and match by `associatedName`. |
| **About / general sections** | Use `imageType: 'scene'` images. |
| **Navigation logo** | Use `imageType: 'logo'` images. |

Build `src/lib/images.ts` by reading `business-data.json` and mapping
`localPath` values to named exports. Example:

```ts
// Read business-data.json images array and map by imageType + associatedName
const data = businessData.images;
const findImage = (type: string, name?: string) =>
  data.find(i => i.imageType === type && (!name || i.associatedName === name))?.localPath;

export const images = {
  hero: findImage('scene') || findImage('banner') || './data/assets/photo-0.jpg',
  logo: findImage('logo') || './data/assets/photo-0.jpg',
  peter: findImage('headshot', 'Peter') || '',
  // ... map each team member by associatedName
};
```

**Banned:**
- ❌ Basic geometric SVGs used as "product images" (a circle is not a photo)
- ❌ Empty colored rectangles as "image placeholders"
- ❌ Skipping images entirely because "the bundler might break"
- ❌ Using stock URLs when scraped images exist in `src/data/assets/`
- ❌ Using `og-image` as hero/section background (it's a logo/icon, not a photo)
- ❌ Guessing image mapping by sequential index (photo-0=hero, photo-1=person1...)
   instead of using `imageType` / `associatedName` metadata

Every section that would logically contain a photograph or illustration in a
real product page **must** have a visual. If the bundler can't handle it, fix
the bundler issue (e.g., use absolute URLs or base64) — do not remove the image.

**Verification:** Every `<Hero />`, `<Features />` (if image+text layout),
`<About />`, and `<Product />` section must contain at least one `<img>` or
high-quality `<svg>` (>20 path/shape elements with gradients or shadows).

### Step 4: Motion Setup

**Mode A note:** If DESIGN.md's "Visual Theme & Atmosphere" or "Component
Stylings" sections describe motion preferences (e.g., "minimal transitions",
"motion-first", "near-zero UI"), adjust easing curves and durations
accordingly. For "minimal/near-zero" styles, reduce all durations by ~40%
and limit reveals to opacity-only. For "motion-first" styles, the defaults
below are appropriate or can be made slightly more expressive.

Create `src/lib/motion.ts` with typed easing and variant constants:

```tsx
export const easeOut: [number, number, number, number] = [0.25, 0.46, 0.45, 0.94];
export const easeStandard: [number, number, number, number] = [0.4, 0, 0.2, 1];
export const easeSpring: [number, number, number, number] = [0.34, 1.56, 0.64, 1];

export const staggerContainer = {
  hidden: {},
  show: { transition: { staggerChildren: 0.08, delayChildren: 0.1 } },
};

export const fadeUp = {
  hidden: { opacity: 0, y: 18 },
  show: { opacity: 1, y: 0, transition: { duration: 0.4, ease: easeOut } },
};
```

Scroll-triggered reveal pattern:
```tsx
import { useRef } from 'react'
import { useInView, motion } from 'framer-motion'
const ref = useRef(null)
const inView = useInView(ref, { once: true, margin: '-80px' })
<motion.div ref={ref} variants={fadeUp} animate={inView ? 'show' : 'hidden'}>
```

**Rules:**
- ❌ NEVER use `linear` easing
- ❌ NEVER use `bounce` or `elastic` easing
- ❌ NEVER animate `width`, `height`, or `padding` — only `transform` and `opacity`
- ❌ NEVER add animation without functional purpose (guide, feedback, comprehension)
- ✅ Respect `prefers-reduced-motion` — disable non-essential animation

**Timing reference:**
| Category       | Duration   | Use                               |
|----------------|------------|-----------------------------------|
| Micro          | 80–200ms   | Hover, focus, toggle, press       |
| Component      | 200–400ms  | Expand/collapse, tab switch       |
| Scene / Page   | 400–800ms  | Route transitions, modals         |
| Stagger offset | 50–150ms   | Sequential list item reveals      |

#### Motion Pattern Library

The base `fadeUp` + `stagger` above are starting points. Each section MUST
select the most appropriate pattern from this library — do NOT reuse
`fadeUp` for every section.

| Pattern | Applicable sections | Effect |
|---------|-------------------|--------|
| `heroTextReveal` | Hero | Words appear sequentially with `opacity + y + blur` transition |
| `heroParallax` | Hero background | `useTransform(scrollY, ...)` shifts background at slower rate than content |
| `useCounter` | Stats / key metrics | Number animates from 0 to target over ~2s using `useSpring` or `requestAnimationFrame` |
| `cardReveal` + `cardHover` | Team, Portfolio, Features | Cards enter with `scale(0.95→1) + opacity`; on hover, `translateY(-4px)` + elevated shadow |
| `sectionReveal` | Section transitions | Entire section fades in with `opacity` only, duration 500–600ms |
| `fadeUp` | Text-heavy sections (About, Contact) | Default vertical reveal — use as fallback only |
| Scroll-driven | Data viz, feature highlights | `useScroll` + `useTransform` for progress-based `scale`/`opacity` tied to scroll position |

**Anti-monotony rule:** No two adjacent sections may use the same primary
motion pattern. If section A uses `fadeUp`, section B must use a different
pattern (e.g., `cardReveal`, `sectionReveal`).

---

### Step 5: Build Sections (Depth-First)

> **When a downstream skill is routed** (artifact-dashboard, artifact-form,
> artifact-interactive): Skip this step — the downstream skill defines its own
> component structure. Steps 1–4 (tokens, typography, images, motion) and
> Step 6 (validation) still apply to all artifact types.

Build sections **one at a time, fully polished, before moving to the next**.
Do NOT scaffold all sections as stubs and fill in later — this leads to
uniformly mediocre output.

**Mode A note:** When building each section, consult DESIGN.md's "Component
Stylings", "Layout Principles", "Depth & Elevation", and "Do's and Don'ts"
sections. These take precedence over the generic component styling rules in
the Design Reference below. For example, if DESIGN.md says "use
shadow-as-border instead of CSS border", follow that even though the default
card styling below specifies `border: 1px solid`.

**Build order** (core sections first):

```
1. <Hero />          — see Hero Section Specification below
2. <Features />      — 3-col grid OR alternating left/right image+text
3. <Pricing />       — 2–3 tiers max, highlight recommended tier
4. <Nav />           — sticky, backdrop-blur, ≤5 nav links + 1 CTA button
5. <Testimonials />  — 2–3 quote cards with author attribution
6. <FAQ />           — shadcn Accordion, 5–8 questions
7. <HowItWorks />    — numbered steps or horizontal timeline
8. <CTA />           — full-width band, single action, high contrast bg
9. <Footer />        — columns of links + legal line, no decorative elements
```

Additional sections from the user's PRD (Product, About, etc.) slot in at the
appropriate position.

#### Hero Section Specification

The hero must answer within 5 seconds: **What is this? Who is it for?
Why should I care?**

**Four required elements** (no more, no less):
1. Headline — benefit-focused, < 60 characters
2. Supporting line — clarifies the headline
3. Visual — image, screenshot, illustration, or typographic treatment
4. CTA — one primary + one optional secondary

**Layout patterns** — pick ONE based on available content:

| Pattern | Layout | Use when | Visual requirement |
|---------|--------|----------|--------------------|
| **Split** | Text 50-60% left + visual right | Product screenshot or mockup available | Product image, app UI, or photo |
| **Full-bleed** | Full-viewport bg image + overlaid text + scrim | High-quality hero photo available (scraped or provided) | Photo with 40%+ low-contrast area for text |
| **Editorial** | Asymmetric grid (25/75 or 60/40), large display headline | Text-heavy / institutional (VC, agency, consulting) — no product photo | Typography IS the visual; pair with data viz, brand mark, or small photo |
| **Centered** | Centered headline stack + CTA, visual below | Single-product launch, event, announcement | Product image or subtle background pattern below headline |
| **Stacked** | Vertical sequence: overline → headline → body → CTA → image | Portfolio, creative, editorial | Featured work or curated visual below fold line |

**Hero background rule:**
- Hero section MUST NOT use a solid-color-only background (pure black,
  pure white, or any flat color fill with no image).
- If the user provided a URL (content sourcer ran), the hero background
  image MUST come from the scraped images (`imageType: 'scene'` or
  `'banner'`). Use it as a full-bleed or partial background with a
  scrim overlay for text contrast.
- If no suitable scraped image exists (no scene/banner images, or all
  are too small / low-quality), switch to a layout pattern that does
  not require a background image: **Editorial** or **Stacked** on a
  light/neutral background (`--color-bg` or `--color-surface`).
- NEVER use a dark/black solid background as a substitute for a
  missing background image.

**Selection rule:** Match pattern to **content available**, not to DESIGN.md's
hero philosophy. If DESIGN.md assumes product photography (e.g., Apple, Tesla)
but the site is text-heavy → use **Editorial** and adapt the color/typography.

**Visual density requirements by background:**

| Background | Minimum visual presence |
|------------|----------------------|
| Image bg (required default) | Scraped scene/banner with scrim (40–70% opacity); text contrast WCAG AA |
| Light / neutral (fallback when no image) | Headline + supporting visual; generous whitespace is fine |

**Headline strategy by industry:**

| Industry | Lead with |
|----------|-----------|
| SaaS / product | Concrete outcome — "Your team meets every deadline" |
| VC / investment | Positioning + conviction — "Early-stage AI. Seed to scale." |
| Agency / studio | Credibility + focus — "Brand and web for growth-stage companies" |
| Restaurant / F&B | Sensory experience — "Authentic Sichuan, reimagined" |
| eCommerce | Value + speed — "Organic skincare, delivered in two days" |

**Typography-driven hero** (when no hero image is available):
- Headline: `clamp(3rem, 6vw, 5rem)` minimum — must command the viewport
- Weight contrast: heavy headline (600–700) + light body (300–400)
- Add a secondary visual anchor: data viz, animated counter, partner logos,
  or editorial photo — do NOT leave the hero with only text
- Text block must feel grounded, not floating in void

**Anti-patterns:**
- ❌ Solid-color-only hero background (dark or light) when scraped scene/banner images are available
- ❌ Dark/black solid background as substitute for missing hero image
- ❌ Geometric primitive SVG (circles + lines) as sole hero visual
- ❌ More than 2 CTAs
- ❌ Generic stock hero that doesn't relate to the business
- ❌ Carousel / slider — always single-frame

Each section lives in its own file under `src/sections/`.
`App.tsx` only imports and stacks sections — no logic there.

**Per-section quality gate — verify BEFORE moving to the next section:**
- [ ] All colors reference `var(--color-*)` or OKLCH values — no raw hex/rgb
- [ ] Typography uses `font-display` / `font-body` classes with scale tokens
- [ ] Images/visuals present where a real product page would have them
- [ ] Interactive elements have at minimum: default, hover, focus, disabled states
- [ ] Spacing uses Tailwind gap/padding utilities — no arbitrary magic numbers

### Step 6: Pre-Bundle Validation

> **This step is mandatory for ALL artifact types** — including those built
> by downstream skills (dashboard, form, interactive, fallback). The visual
> quality gate applies universally.

Run the full anti-slop checklist. **Every item must pass. If any fails, fix
it before bundling.**

```bash
echo "=== Anti-Slop Validation ==="

# 1. OKLCH tokens present
OKLCH_COUNT=$(grep -c "oklch" src/index.css)
[ "$OKLCH_COUNT" -ge 8 ] && echo "✅ OKLCH tokens: $OKLCH_COUNT" || echo "❌ OKLCH tokens: only $OKLCH_COUNT (need ≥8)"

# 2. No raw hex in section code (excluding SVG icon paths)
HEX_HITS=$(grep -rn '#[0-9a-fA-F]\{3,8\}' src/sections/ | grep -v '\.svg' | wc -l)
[ "$HEX_HITS" -eq 0 ] && echo "✅ No raw hex in sections" || echo "❌ Found $HEX_HITS raw hex values in sections"

# 3. No banned fonts
BANNED=$(grep -rn 'Inter\|Roboto\|Arial\|"Open Sans"\|Helvetica' tailwind.config.js src/ | wc -l)
[ "$BANNED" -eq 0 ] && echo "✅ No banned fonts" || echo "❌ Found $BANNED banned font references"

# 4. Font families configured
grep -q '"display"\|font-display\|display:' tailwind.config.js && echo "✅ Display font configured" || echo "❌ Missing display font"
grep -q '"body"\|font-body\|body:' tailwind.config.js && echo "✅ Body font configured" || echo "❌ Missing body font"

# 5. CTA section exists
[ -f "src/sections/CTA.tsx" ] || [ -f "src/sections/Cta.tsx" ] && echo "✅ CTA section exists" || echo "❌ Missing CTA section"

# 6. Images present in key sections
for section in Hero Product About; do
  FILE=$(find src/sections -iname "${section}*" -type f | head -1)
  if [ -n "$FILE" ]; then
    IMG_COUNT=$(grep -c '<img\|<svg' "$FILE")
    [ "$IMG_COUNT" -ge 1 ] && echo "✅ $section has visuals ($IMG_COUNT)" || echo "❌ $section has no visuals"
  fi
done

# 7. Content-sourced image integrity (only when scraper was used)
if [ -f "src/data/business-data.json" ]; then
  SCRAPED_IMGS=$(ls src/data/assets/photo-*.{jpg,png,webp} 2>/dev/null | wc -l | tr -d ' ')
  if [ "$SCRAPED_IMGS" -gt 0 ]; then
    STOCK_IN_SECTIONS=$(grep -rn 'unsplash\.com\|pexels\.com\|pixabay\.com' src/sections/ | wc -l | tr -d ' ')
    [ "$STOCK_IN_SECTIONS" -eq 0 ] \
      && echo "✅ No stock image URLs (using scraped images)" \
      || echo "❌ Found $STOCK_IN_SECTIONS stock image URLs — replace with scraped images from src/data/assets/"
  else
    echo "ℹ️  No scraped images available — stock URLs acceptable"
  fi

  # 8. OG image not used as hero background
  node -e "
  const d=JSON.parse(require('fs').readFileSync('src/data/business-data.json','utf8'));
  const ogImgs=d.images.filter(i=>i.imageType==='og-image').map(i=>i.localPath).filter(Boolean);
  if(ogImgs.length===0){console.log('ℹ️  No OG images to check');process.exit(0);}
  const fs=require('fs');
  const heroFile=fs.readdirSync('src/sections/').find(f=>/hero/i.test(f));
  if(!heroFile){console.log('ℹ️  No hero section to check');process.exit(0);}
  const heroCode=fs.readFileSync('src/sections/'+heroFile,'utf8');
  const ogUsed=ogImgs.some(p=>{const fname=p.split('/').pop();return heroCode.includes(fname);});
  if(ogUsed) console.log('❌ OG image used as hero background — replace with scene/banner image');
  else console.log('✅ OG image not misused as hero background');
  " 2>/dev/null

  # 9. Team image mapping uses structured data (not sequential guessing)
  node -e "
  const d=JSON.parse(require('fs').readFileSync('src/data/business-data.json','utf8'));
  const team=d.team||[];
  if(team.length===0){console.log('ℹ️  No team data to check');process.exit(0);}
  const withUrl=team.filter(m=>m.imageUrl);
  if(withUrl.length===0){console.log('⚠️  Team members have no imageUrl — check scraper');process.exit(0);}
  const fs=require('fs');
  const imgFile=fs.readdirSync('src/lib/').find(f=>/image/i.test(f));
  if(!imgFile){console.log('⚠️  No image mapping file found in src/lib/');process.exit(0);}
  const imgCode=fs.readFileSync('src/lib/'+imgFile,'utf8');
  const hasAssocMapping=imgCode.includes('associatedName')||imgCode.includes('imageType')||imgCode.includes('imageUrl');
  if(hasAssocMapping) console.log('✅ Image mapping uses structured data');
  else console.log('⚠️  Image mapping may use sequential guessing — verify manually');
  " 2>/dev/null
fi

echo "=== End Validation ==="
```

**Manual checklist (review visually):**
- [ ] Hero text is left-aligned or asymmetric — NOT centered paragraph blocks
- [ ] No purple, teal, or rainbow gradients used as decorative backgrounds
  (unless DESIGN.md explicitly uses them, e.g., Stripe's signature gradients)
- [ ] Border-radius varies by component type — not uniform `rounded-xl` everywhere
- [ ] Every interactive element has at least: default, hover, focus, disabled states
- [ ] Animations use `transform` / `opacity` only
- [ ] No "card soup" — cards not nested inside cards
- [ ] Shadows are tinted toward primary hue, not flat gray

**Mode A additional check:** If DESIGN.md has a "Do's and Don'ts" section,
verify you have not violated any of its "Don't" rules. DESIGN.md's
do's/don'ts override this skill's generic anti-slop rules when they conflict
(e.g., DESIGN.md may allow centered hero text for certain brand aesthetics).

Only after all checks pass → run `bundle-artifact.sh`.

---

## Design Reference

The following sections are reference material for the design contract. They
are enforced through Steps 1–6 above.

### Spacing & Layout

```css
--space-1:  0.25rem;  /* 4px  — icon padding, inline gaps */
--space-2:  0.5rem;   /* 8px  — tight element spacing */
--space-3:  0.75rem;  /* 12px — form field gaps */
--space-4:  1rem;     /* 16px — default component padding */
--space-6:  1.5rem;   /* 24px — section internal padding */
--space-8:  2rem;     /* 32px — between related sections */
--space-12: 3rem;     /* 48px — between major sections */
--space-16: 4rem;     /* 64px — page-level vertical rhythm */
```

**Grid**: Desktop 12-col fluid, max-width 1280px, gap `var(--space-6)` / Tablet 6–8 col / Mobile 4 col

- ✅ Use `gap` in flex/grid instead of margin for layout spacing
- ✅ Prefer `@container` queries over `@media` for component-level responsiveness
- ✅ Fluid spacing with `clamp()` so layouts breathe on large screens
- ❌ NEVER equal padding everywhere — spacing must reflect content hierarchy
- ❌ NEVER center-align everything — left-align text, use asymmetry for visual interest
- ❌ NEVER nest cards inside cards ("card soup")

### Component Styling

**Buttons**
- Border-radius: `0.5rem` (8px) standard, `9999px` pill
- Primary: `bg: var(--color-primary)`, `text: white`, hover → darken 1 OKLCH step + subtle shadow
- Secondary: `border: 1px solid var(--color-primary)`, hover → light primary-tinted bg
- Ghost: transparent bg, hover → subtle bg tint
- Disabled: 40–50% opacity, no shadow, no hover, `cursor: not-allowed`
- Focus: 2px focus ring, offset 2px, high-contrast, visible on keyboard nav

**Cards & Surfaces**
- Border-radius: `0.75rem` cards, `1rem` dialogs
- Background: `var(--color-surface)`, border: `1px solid var(--color-border)`
- Shadow (elevated only): `0 1px 3px oklch(0% 0 0 / 0.08), 0 4px 12px oklch(0% 0 0 / 0.04)`
- Internal padding: `var(--space-6)`, internal gap: `var(--space-3)` to `var(--space-4)`

**Inputs & Forms**
- Default: `bg: var(--color-bg)`, `border: 1px solid var(--color-border)`, border-radius: `0.5rem`
- Focus: border → `var(--color-primary)`, 2px outline
- Error: border → `var(--color-error)`, helper text in error color + inline icon
- Labels: always visible (no placeholder-only labels), `font-weight: 500`

**Required Interaction States — every interactive component must implement:**
1. **Default** — resting state
2. **Hover** — subtle visual feedback
3. **Focus** — keyboard-accessible indicator
4. **Active/Pressed** — confirmation of interaction
5. **Disabled** — clearly non-interactive

States 6–8 below are required only for components where they are semantically
relevant (e.g., forms, data-loading views):
6. **Loading** — skeleton or spinner, never blank
7. **Empty** — helpful message + action, never blank
8. **Error** — clear message + recovery path

### Accessibility

- Semantic HTML: `<button>` for actions, `<a>` for navigation, `<label>` bound to `<input>`
- Keyboard navigable: all interactive elements reachable via Tab, operable via Enter/Space
- Focus management: dialogs trap focus, menus return focus on close
- Touch targets: minimum 44×44px
- Contrast: WCAG 2.2 AA minimum (4.5:1 body text, 3:1 large text / UI elements)
- Color is never the sole indicator of state

### Typography in Practice

```tsx
// Hero headline — fluid, expressive
<h1 className="font-display text-[clamp(2.5rem,5vw,3.5rem)] font-bold leading-[1.1] tracking-tight">
  {headline}
</h1>

// Section title
<h2 className="font-display text-[clamp(1.5rem,3vw,2rem)] font-semibold leading-[1.2]">
  {title}
</h2>

// Body paragraph — constrained width for readability
<p className="font-body text-base leading-[1.6] max-w-[65ch] text-[var(--color-text-secondary)]">
  {copy}
</p>
```
