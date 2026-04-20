---
name: website-generator
description: >-
  Generate web artifacts and HTML pages using React, Tailwind CSS, and shadcn/ui.
  Handles landing pages, dashboards, forms, surveys, interactive tools, games,
  and calculators. Optionally sources content from public URLs (Google Business
  Profile, Yelp, etc.) via headless browser. Acts as a multi-step router:
  optionally sources content, initializes via web-artifacts-builder, then layers
  on a specialized sub-skill based on intent. Use this as the single entry point
  for any web artifact or website generation request.
---

# Website Generator

**Do not write any code yet.** Follow all steps below in order.

---

## Priority Rule

**Skill design principles are the highest priority.** When the user's query or PRD
conflicts with a skill's design contract (color system, typography rules,
anti-slop rules, layout patterns, etc.), always follow the skill. The skill
defines *how* to build — the user defines *what* to build. Never silently
downgrade or skip a skill requirement because it seems hard or conflicts with
the user's spec. If a conflict is genuinely irreconcilable, ask the user
before proceeding.

---

## Step 0 (Conditional): Content Sourcing

If the user provides a **URL** as a content source (Google Business Profile,
Yelp listing, Instagram profile, company website, etc.), read
`skills/content-sourcer/SKILL.md` and follow its Execution Protocol to extract
business information and download image assets **before** initializing the
project.

**Trigger signals**: any URL in the user's request alongside a website
generation ask — e.g., "here's my Google Business profile", "use info from
this Yelp page", "grab content from my website".

**Skip this step** if the user provides no URLs and gives all content directly
in the chat (business name, description, images in workspace, etc.).

The extracted data (JSON + images) will be consumed by downstream skills in
Steps 2–3.

> ⚠️ **Content Sourcer is mandatory when a URL is provided.** You must run
> the Playwright-based scraper — do not substitute with `WebFetch`, web
> search, or manually assembled data. Read `skills/content-sourcer/SKILL.md`
> carefully: it contains Critical Rules that forbid these shortcuts.

---

## Step 1 (Always): Initialize with web-artifacts-builder

Read `skills/web-artifacts-builder/SKILL.md` and follow its instructions to
initialize the project and prepare for bundling. This step is required for
every request regardless of intent.

---

## Step 1.5 (Conditional): Content Sourcing Gate Check

> **Run this gate only if Step 0 was triggered** (user provided a URL).
> If Step 0 was skipped, skip this gate too.

Before proceeding to Step 2, verify that the content sourcer produced
valid output. Run:

```bash
echo "=== Content Sourcing Gate Check ==="
DATA_DIR="<PROJECT_DIR>/src/data"

# 1. business-data.json exists
[ -f "$DATA_DIR/business-data.json" ] \
  && echo "✅ business-data.json exists" \
  || echo "❌ BLOCKING: business-data.json missing — go back and run the scraper"

# 2. File was produced by the scraper (has scrapedAt field)
grep -q '"scrapedAt"' "$DATA_DIR/business-data.json" 2>/dev/null \
  && echo "✅ Produced by scraper" \
  || echo "❌ BLOCKING: File missing scrapedAt — was it hand-written? Re-run scraper"

# 3. Check for downloaded images
IMG_COUNT=$(ls "$DATA_DIR/assets/"photo-*.{jpg,png,webp} 2>/dev/null | wc -l | tr -d ' ')
echo "ℹ️  Downloaded images: $IMG_COUNT"

# 4. Content depth — section count across main page + sub-pages
SECTION_COUNT=$(node -e "
const d=JSON.parse(require('fs').readFileSync('$DATA_DIR/business-data.json','utf8'));
let c=(d.sections||[]).length;
for(const sp of d.subPages||[]) c+=(sp.sections||[]).length;
console.log(c)" 2>/dev/null || echo "0")
echo "ℹ️  Total sections extracted: $SECTION_COUNT"

# 5. Section type coverage
SECTION_TYPES=$(node -e "
const d=JSON.parse(require('fs').readFileSync('$DATA_DIR/business-data.json','utf8'));
const all=[...(d.sections||[])];
for(const sp of d.subPages||[]) all.push(...(sp.sections||[]));
const types=[...new Set(all.map(s=>s.sectionType).filter(t=>t&&t!=='unknown'))];
console.log(types.join(', ')||'(none)')" 2>/dev/null || echo "(error)")
echo "ℹ️  Section types: $SECTION_TYPES"

# 6. Sub-pages and sitemap info
SUB_PAGES=$(node -e "const d=JSON.parse(require('fs').readFileSync('$DATA_DIR/business-data.json','utf8')); console.log((d.subPages||[]).map(p=>p.path).join(', ')||'(none)')" 2>/dev/null || echo "(none)")
echo "ℹ️  Sub-pages crawled: $SUB_PAGES"

# 7. Content quality summary
node -e "
const d=JSON.parse(require('fs').readFileSync('$DATA_DIR/business-data.json','utf8'));
const warnings=[];
if((d.description||'').length<50) warnings.push('short description');
let sc=(d.sections||[]).length;
for(const sp of d.subPages||[]) sc+=(sp.sections||[]).length;
if(sc<3) warnings.push('few sections ('+sc+')');
if((d.images||[]).filter(i=>i.localPath).length===0) warnings.push('no downloaded images');
if(warnings.length>0) console.log('⚠️  Content quality concerns: '+warnings.join(', '));
else console.log('✅ Content quality looks good');
" 2>/dev/null

# 8. Image classification coverage
node -e "
const d=JSON.parse(require('fs').readFileSync('$DATA_DIR/business-data.json','utf8'));
const imgs=d.images||[];
const types={};
for(const img of imgs){const t=img.imageType||'unclassified';types[t]=(types[t]||0)+1;}
const summary=Object.entries(types).map(([k,v])=>k+':'+v).join(', ');
console.log('ℹ️  Image classification: '+summary);
const hasOg=imgs.some(i=>i.imageType==='og-image');
const hasScene=imgs.some(i=>i.imageType==='scene'||i.imageType==='banner');
if(hasOg&&!hasScene) console.log('⚠️  Only OG image found — no suitable hero background available, use typography-driven hero');
" 2>/dev/null

# 9. Team member photo mapping integrity
node -e "
const d=JSON.parse(require('fs').readFileSync('$DATA_DIR/business-data.json','utf8'));
const team=d.team||[];
if(team.length===0){console.log('ℹ️  No team members');process.exit(0);}
const withPhoto=team.filter(m=>m.imageUrl);
console.log('ℹ️  Team: '+team.length+' members, '+withPhoto.length+' with direct photo URL');
const headshots=(d.images||[]).filter(i=>i.imageType==='headshot');
const namedHeadshots=headshots.filter(i=>i.associatedName);
console.log('ℹ️  Headshot images: '+headshots.length+' total, '+namedHeadshots.length+' with name');
if(withPhoto.length<team.length&&namedHeadshots.length<team.length)
  console.log('⚠️  Not all team members have mapped photos — verify image assignment in build step');
" 2>/dev/null

echo "=== End Gate Check ==="
```

**If any ❌ BLOCKING check fails**: do NOT proceed. Go back to Step 0 and
run the content-sourcer scraper properly. See
`skills/content-sourcer/SKILL.md` → Critical Rules.

**If images are 0 but other checks pass**: proceed, but ensure downstream
sections use stock imagery AND the user has been informed that scraped
images were unavailable.

**If content quality concerns appear (⚠️)**: proceed, but inform the user
that scraped content was limited. When building sections, supplement sparse
data with reasonable defaults rather than generating empty blocks. Tell the
user which sections used AI-generated placeholder content vs. scraped data.

---

## Step 2 (Always): Visual Design Layer

After the project is initialized (and content gate checked, if applicable),
establish the visual design foundation. Read `skills/artifact-visual/SKILL.md`
and follow its Execution Protocol steps in order.

This step is **mandatory for every artifact type**. `artifact-visual` handles:
- Design system selection (from its `design-system-index.md`)
- OKLCH color tokens
- Typography configuration
- Image strategy
- Motion setup

The visual tokens and rules established here apply to **all** downstream
artifact types — dashboards, forms, interactive tools, and generic landing
pages alike.

> When a downstream skill is routed in Step 3, `artifact-visual`'s Step 5
> (Build Sections) is skipped — the downstream skill defines its own
> component structure. But Steps 1–4 (tokens, typography, images, motion)
> and Step 6 (pre-bundle validation) are always enforced.

---

## Step 2.5 (Conditional): Image Review

> **Run this step only if Step 0 was triggered** (user provided a URL) AND
> images were downloaded. Skip if no images exist.

Before building sections, review the scraped image classification to catch
misassignment early. This prevents the most common visual defects (wrong
hero image, wrong team photos).

```bash
echo "=== Image Review ==="
node -e "
const d=JSON.parse(require('fs').readFileSync('<PROJECT_DIR>/src/data/business-data.json','utf8'));
const imgs=(d.images||[]).filter(i=>i.localPath);
console.log('\\n📸 Downloaded images with classification:\\n');
for(const img of imgs){
  const file=img.localPath.split('/').pop();
  const type=(img.imageType||'unknown').padEnd(14);
  const sec=(img.sectionType||'-').padEnd(12);
  const name=img.associatedName||'-';
  console.log('  '+file.padEnd(18)+type+sec+name);
}
console.log('');
const team=d.team||[];
if(team.length>0){
  console.log('👥 Team member → photo mapping:');
  for(const m of team){
    const photo=m.imageUrl?'✅ has imageUrl':'❌ no imageUrl';
    console.log('  '+m.name.padEnd(25)+photo);
  }
  console.log('');
}
const heroCandidate=imgs.find(i=>i.imageType==='scene'||i.imageType==='banner');
const ogOnly=!heroCandidate&&imgs.some(i=>i.imageType==='og-image');
if(heroCandidate) console.log('🖼️  Hero candidate: '+heroCandidate.localPath.split('/').pop()+' ('+heroCandidate.imageType+')');
else if(ogOnly) console.log('⚠️  No scene/banner images — only OG image available. Use typography-driven hero, NOT the OG image as background.');
else console.log('⚠️  No hero image candidate found — use typography-driven hero or stock image.');
" 2>/dev/null
echo "=== End Image Review ==="
```

Use this information when building `src/lib/images.ts` in Step 3. The
image mapping must be driven by `imageType` and `associatedName`, not
sequential file index assumptions.

---

## Step 3 (Intent-based): Follow a specialized skill's Execution Protocol

After completing Step 2 (visual design layer), analyze the user's request
and determine whether a specialized interaction skill applies. If it does,
read that skill's SKILL.md and **follow its Execution Protocol steps in
order, completing each step before moving to the next**. Do not cherry-pick
patterns or skip steps — the sub-skill's protocol is designed as a mandatory
sequence where each step depends on the previous one.

All visual tokens (OKLCH colors, typography, motion) from Step 2 carry
forward — downstream skills define **component structure and interaction
patterns**, not visual identity.

If the sub-skill has a Pre-Bundle Validation step, you **must** run it and
fix all failures before bundling.

### Intent routing table

| User wants... | Signal words / patterns | Read this skill |
|---|---|---|
| Data visualization / analytics | dashboard, analytics, metrics, KPIs, charts, graphs, admin panel, reporting | `skills/artifact-dashboard/SKILL.md` |
| A form or survey | form, survey, questionnaire, registration, onboarding, checkout, wizard, sign-up | `skills/artifact-form/SKILL.md` |
| An interactive tool | calculator, converter, generator, quiz, game, configurator, timer, editor | `skills/artifact-interactive/SKILL.md` |
| **Everything else** | landing page, marketing page, portfolio, company site, vague description, "make me a website" | `skills/fallback/SKILL.md` |

### Fallback (default path)

When the request doesn't match dashboard, form, or interactive, read
`skills/fallback/SKILL.md`. This provides the default page structure
(Nav + Hero + Features + CTA + Footer). The visual quality is already
guaranteed by Step 2 — `fallback` only decides *what sections to build*.

---

## Routing Examples

| Request | Step 2 design system | Step 3 skill |
|---|---|---|
| "Build a SaaS landing page" | `vercel` or `linear.app` | `fallback` |
| "Create a sales dashboard with monthly revenue charts" | `sentry` or `posthog` | `artifact-dashboard` |
| "Make a multi-step onboarding form with Zod validation" | `cal` or `intercom` | `artifact-form` |
| "Build a BMI calculator with instant feedback" | `notion` or `wise` | `artifact-interactive` |
| "Make me a cool website" | `framer` or `figma` | `fallback` |
| "A product page with a contact form at the bottom" | `stripe` or `shopify` | `fallback` |
| "A dashboard with a filter panel" | `sentry` or `semrush` | `artifact-dashboard` |
| "做一个 VC 机构官网" | `spacex` or `lamborghini` | `fallback` |
| "做一个火锅餐厅网站" | `airbnb` | `fallback` |
| "Build a dark premium portfolio" | `superhuman` or `tesla` | `fallback` |

### Compound requests

Pick the **dominant interaction** — what the user interacts with most:
- Landing page + contact form → `fallback` (borrow form patterns inline)
- Dashboard + settings form → `artifact-dashboard`
- Multi-step form + result chart → `artifact-form`
- Quiz tool + score visualization → `artifact-interactive`

Borrow patterns from non-primary skills inline as needed once the primary skill
has been read.

### Content-sourcing scenarios

When the user provides a URL, Step 0 runs first, Step 2 selects a design
system, then Step 3 routing proceeds as usual based on intent:

| Request | Step 0 | Step 2 design | Step 3 skill |
|---|---|---|---|
| "Build a website for my restaurant, here's my Google profile: ..." | `content-sourcer` | `airbnb` | `fallback` |
| "Redesign my Yelp page as a proper landing page" | `content-sourcer` | (match industry) | `fallback` |
| "Make a dashboard from my business reviews" | `content-sourcer` | `posthog` | `artifact-dashboard` |
| "Here's my IG, build me a portfolio" | `content-sourcer` (restricted) | `framer` or `clay` | `fallback` |

---

## Step 4 (Always): Validate and Bundle

> **If Step 0 was run**, the project's `src/data/` directory now contains
> `business-data.json` and an `assets/` folder with downloaded images.
> Components built in Step 3 should import this data rather than using
> placeholder content.

After completing the sub-skill's Execution Protocol:

1. **Run the sub-skill's Pre-Bundle Validation script** (if it has one).
   Fix every failure before proceeding — do not bundle with known violations.
2. **If Step 0 was run**, run the content-sourcing image check below.
   Fix every ❌ before proceeding.
3. **Run `scripts/bundle-artifact.sh`** to produce the single-file HTML artifact.
4. **Share the bundled file** with the user.

### Content-Sourcing Image Check (if Step 0 was run)

```bash
echo "=== Content Source Image Check ==="

# 1. No stock placeholder URLs in section code when scraper downloaded images
IMG_COUNT=$(ls src/data/assets/photo-*.{jpg,png,webp} 2>/dev/null | wc -l | tr -d ' ')
if [ "$IMG_COUNT" -gt 0 ]; then
  STOCK_URLS=$(grep -rn 'unsplash\.com\|pexels\.com\|pixabay\.com\|stockphoto' src/sections/ | wc -l | tr -d ' ')
  [ "$STOCK_URLS" -eq 0 ] \
    && echo "✅ No stock URLs in sections (using scraped images)" \
    || echo "❌ Found $STOCK_URLS stock image URLs — replace with scraped images from src/data/assets/"
fi

# 2. Sections reference scraped data, not hardcoded content
IMPORTS_DATA=$(grep -rn "business-data\|businessData\|siteData" src/sections/ src/App.tsx | wc -l | tr -d ' ')
[ "$IMPORTS_DATA" -gt 0 ] \
  && echo "✅ Components reference scraped data ($IMPORTS_DATA imports)" \
  || echo "⚠️  No scraped data imports found — sections may use hardcoded content"

# 3. Scraped section types are reflected in built sections
node -e "
const d=JSON.parse(require('fs').readFileSync('src/data/business-data.json','utf8'));
const all=[...(d.sections||[])];
for(const sp of d.subPages||[]) all.push(...(sp.sections||[]));
const scraped=new Set(all.map(s=>s.sectionType).filter(t=>t&&t!=='unknown'));
const fs=require('fs'), built=fs.readdirSync('src/sections/').map(f=>f.replace(/\.tsx$/,'').toLowerCase());
const missed=[...scraped].filter(t=>!built.some(b=>b.includes(t)));
if(missed.length>0) console.log('⚠️  Scraped section types not represented in build: '+missed.join(', '));
else console.log('✅ All key scraped section types have corresponding built sections');
" 2>/dev/null

echo "=== End Content Source Image Check ==="
```

Never bundle without running validation first. If you skipped Step 3 (fallback
path), still verify basic quality: no blank screens, no console-only errors,
responsive layout, functional interactions.
