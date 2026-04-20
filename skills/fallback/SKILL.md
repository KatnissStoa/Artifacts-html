---
name: fallback
description: >-
  Default page structure for landing pages, marketing pages, portfolios, company
  sites, and any request that doesn't match dashboard, form, or interactive.
  Provides section selection and build order on top of the visual foundation
  already established by artifact-visual (Step 2 of the main router).
---

# Fallback: Default Page Structure

This skill is reached when Step 3 of the main router selects the fallback
path — either because the request is a standard landing/marketing page, or
because no specialized interaction skill (dashboard, form, interactive) was
matched.

> **Visual design is already handled.** `artifact-visual` (Step 2) has
> configured OKLCH tokens, typography, image strategy, and motion. This skill
> only decides *what sections to build* and in *what order*. All sections must
> follow the visual rules from Step 2.

## Step 1: Attempt One Clarifying Question (Optional)

If a single question would meaningfully change the output, ask it before building:

> "What's the main purpose — a landing/marketing page, a data dashboard, a form
> or survey, or an interactive tool?"

**Skip this** if any of the following are true:
- You can infer a reasonable intent from context (topic, examples, domain)
- The user seems to want quick output, not a back-and-forth
- Any reasonable interpretation would produce something useful

## Step 2: Build Sections

Use `artifact-visual`'s Step 5 (Build Sections) as the build protocol —
follow its section ordering, hero specification, per-section quality gate,
and depth-first build approach.

**Default section set** (when the user's request doesn't specify sections):

```
1. <Nav />           — sticky, backdrop-blur, ≤5 nav links + 1 CTA button
2. <Hero />          — see artifact-visual's Hero Section Specification
3. <Features />      — 3-col grid OR alternating left/right image+text
4. <CTA />           — full-width band, single action, high contrast bg
5. <Footer />        — columns of links + legal line, no decorative elements
```

Additional sections from the user's PRD (Pricing, Testimonials, FAQ,
HowItWorks, About, Team, etc.) slot in at the appropriate position per
`artifact-visual`'s build order.

**File layout:**
```
src/
  App.tsx          → imports and stacks all sections, no logic
  sections/
    Nav.tsx
    Hero.tsx
    Features.tsx
    CTA.tsx
    Footer.tsx
```

## Step 3: Offer to Pivot

After presenting the artifact, always add:

> "I built this as a general landing page based on your description. If you had
> something else in mind — a data dashboard, a form/survey, or an interactive
> tool — let me know and I'll rebuild it with the right approach."
