---
name: content-sourcer
description: >-
  Extract business information and images from public URLs (Google Business
  Profile, Yelp, general websites) using Playwright headless browser. Outputs
  structured JSON data and downloaded image assets for downstream skills to
  consume. Gracefully degrades for login-required platforms (Instagram, etc.)
  by informing the user that data may be incomplete.
---

# Content Sourcer

> This skill runs **before** the main build pipeline. It gathers content from
> user-provided URLs and produces a structured data file + local image assets
> that downstream skills (`artifact-visual`, `fallback`, etc.) can reference.

> **Working directory**: Run every shell command below from the **repository root** — the directory that contains the root `SKILL.md` and the `skills/` folder. Paths such as `skills/content-sourcer/scripts/...` are relative to that root.

## When to Trigger

Activate this skill when the user provides **any URL** as a content source for
the website they want to build. Common signals:

- "here's my Google Business profile: ..."
- "use info from this Yelp page: ..."
- "grab content from my Instagram: ..."
- "here's my website, redesign it: ..."
- Any URL alongside a website generation request

## Critical Rules

> **These rules are non-negotiable. Violating any of them produces incorrect
> output that undermines the entire content-sourcing purpose.**

1. **ALWAYS run the Playwright scraper** (`scrape-profile.ts`). Do NOT
   substitute with `WebFetch`, `fetch()`, web search results, or manually
   assembled JSON. The scraper uses a headless Chromium browser that handles
   JavaScript rendering, redirects, cookie consent dialogs, and lazy-loaded
   images — none of which simple HTTP requests can handle.

2. **NEVER fabricate or hand-write `business-data.json`**. The file must be
   produced by `scrape-profile.ts`. If you find yourself writing JSON with
   business fields by hand, you have skipped the scraper — go back and run it.

3. **NEVER use Unsplash, Pexels, or other stock-photo URLs as a substitute
   for scraped images** when this skill was triggered. Downstream components
   must reference images downloaded by the scraper to `src/data/assets/`.
   Stock photos are only acceptable when the scraper ran but found zero
   downloadable images AND the user has been informed.

4. **If the scraper fails**, inform the user with the exact error, then ask
   them to provide data manually. Do NOT silently fall back to web-search
   results and pretend the scraper ran.

---

## Execution Protocol

Complete each step in order. **Do not skip or reorder steps.**

### Step 0: Install Dependencies

Run the setup script once per session (idempotent — safe to re-run):

```bash
bash skills/content-sourcer/scripts/setup-scraper.sh
```

This installs Playwright, Chromium, tsx, and Cheerio. You **must** run this
before Step 3 — do not skip it even if you think dependencies are installed.

### Step 1: Classify the URL

Determine the URL type and accessibility level:

| URL Pattern | Source Type | Access Level |
|---|---|---|
| `share.google/`, `google.com/maps`, `maps.app.goo.gl`, `g.co/` | Google Business Profile | Public — full extraction |
| `yelp.com/biz/` | Yelp Business | Public — full extraction |
| `tripadvisor.com/` | TripAdvisor | Public — full extraction |
| `facebook.com/` (page, not personal profile) | Facebook Page | Public — partial extraction |
| `instagram.com/`, `instagr.am/` | Instagram | **Restricted** — login required |
| `xiaohongshu.com/`, `xhslink.com/` | Xiaohongshu | **Restricted** — login required |
| Other | Generic Website | Public — best-effort extraction |

> **Note on `share.google/` URLs**: These short-links redirect through
> multiple hops before landing on the Google Maps panel. Simple HTTP fetch
> (`WebFetch`, `curl`, etc.) will return a redirect stub or error page —
> this is expected and does NOT mean the URL is broken. The Playwright
> scraper handles this correctly via its headless browser. **Always proceed
> to Step 3.**

### Step 2: Handle Restricted URLs

If the URL is classified as **Restricted**, immediately inform the user:

> **Access restricted**: This platform requires login to view full content.
> I can attempt to extract publicly visible information, but data (especially
> images) may be incomplete or unavailable.
>
> For best results, you can:
> 1. Save images from the profile manually to a local `assets/` folder
> 2. Provide business details (name, description, hours) directly in the chat
>
> I'll proceed with whatever public data is accessible.

Then continue to Step 3 anyway — attempt extraction with the understanding
that results may be partial.

### Step 3: Run the Scraper (MANDATORY)

> ⚠️ **This step is mandatory.** You MUST execute the command below. Do not
> skip it, do not replace it with web search, do not hand-write the output
> file. See Critical Rules above.

Execute the scraping script with the classified URL:

```bash
cd skills/content-sourcer/scripts && npx tsx scrape-profile.ts \
  --url "<USER_URL>" \
  --type "<SOURCE_TYPE>" \
  --output-dir "<ABSOLUTE_PATH_TO_PROJECT>/src/data"
```

The script produces:
- `<output-dir>/business-data.json` — structured business information
- `<output-dir>/assets/` — downloaded images (if available)

If the script exits with an error, see **Fallback Behavior** at the bottom
of this document. Do NOT silently work around the failure.

### Step 4: Verify Scraper Output (BLOCKING)

Run the verification script below. **All checks must pass before
proceeding.** If any check fails, the scraper did not run properly — go
back to Step 3 and fix the issue.

```bash
echo "=== Content Sourcer Verification ==="
OUTPUT_DIR="<ABSOLUTE_PATH_TO_PROJECT>/src/data"

# 1. business-data.json exists and was produced by the scraper
[ -f "$OUTPUT_DIR/business-data.json" ] \
  && echo "✅ business-data.json exists" \
  || { echo "❌ business-data.json missing — scraper did not run"; exit 1; }

# 2. File contains scrapedAt timestamp (proves scraper wrote it)
grep -q '"scrapedAt"' "$OUTPUT_DIR/business-data.json" \
  && echo "✅ Has scrapedAt timestamp" \
  || echo "⚠️  Missing scrapedAt — file may be hand-written"

# 3. Check for downloaded images
IMG_COUNT=$(ls "$OUTPUT_DIR/assets/"photo-*.{jpg,png,webp} 2>/dev/null | wc -l | tr -d ' ')
[ "$IMG_COUNT" -gt 0 ] \
  && echo "✅ Downloaded images: $IMG_COUNT" \
  || echo "⚠️  No downloaded images in assets/ — inform user, use stock only as last resort"

# 4. Business name is non-empty
NAME=$(grep -o '"name"[[:space:]]*:[[:space:]]*"[^"]*"' "$OUTPUT_DIR/business-data.json" | head -1)
[ -n "$NAME" ] \
  && echo "✅ Business name found: $NAME" \
  || echo "⚠️  Business name empty — ask user to provide"

# 5. Content depth: description length
DESC_LEN=$(node -e "const d=JSON.parse(require('fs').readFileSync('$OUTPUT_DIR/business-data.json','utf8')); console.log((d.description||'').length)" 2>/dev/null || echo "0")
[ "$DESC_LEN" -gt 50 ] \
  && echo "✅ Description length: ${DESC_LEN} chars" \
  || echo "⚠️  Description too short (${DESC_LEN} chars) — generation may lack context"

# 6. Section count (main page + sub-pages combined)
SECTION_COUNT=$(node -e "
const d=JSON.parse(require('fs').readFileSync('$OUTPUT_DIR/business-data.json','utf8'));
let c=(d.sections||[]).length;
for(const sp of d.subPages||[]) c+=(sp.sections||[]).length;
console.log(c)" 2>/dev/null || echo "0")
echo "ℹ️  Extracted sections: $SECTION_COUNT"
[ "$SECTION_COUNT" -ge 3 ] \
  && echo "✅ Sufficient section content" \
  || echo "⚠️  Few sections extracted — page may lack content depth, consider asking user for more info"

# 7. Section type coverage
SECTION_TYPES=$(node -e "
const d=JSON.parse(require('fs').readFileSync('$OUTPUT_DIR/business-data.json','utf8'));
const all=[...(d.sections||[])];
for(const sp of d.subPages||[]) all.push(...(sp.sections||[]));
const types=[...new Set(all.map(s=>s.sectionType).filter(t=>t&&t!=='unknown'))];
console.log(types.join(', ') || '(none)')" 2>/dev/null || echo "(error)")
echo "ℹ️  Section types found: $SECTION_TYPES"

# 8. Sub-pages crawled
SUB_PAGE_COUNT=$(node -e "const d=JSON.parse(require('fs').readFileSync('$OUTPUT_DIR/business-data.json','utf8')); console.log((d.subPages||[]).length)" 2>/dev/null || echo "0")
echo "ℹ️  Sub-pages crawled: $SUB_PAGE_COUNT"

# 9. Sitemap page count
SITEMAP_PAGES=$(node -e "const d=JSON.parse(require('fs').readFileSync('$OUTPUT_DIR/business-data.json','utf8')); console.log(d.sitemapPageCount ?? 'N/A')" 2>/dev/null || echo "N/A")
echo "ℹ️  Sitemap total pages: $SITEMAP_PAGES"

# 10. Image classification coverage
node -e "
const d=JSON.parse(require('fs').readFileSync('$OUTPUT_DIR/business-data.json','utf8'));
const imgs=d.images||[];
const types={};
for(const img of imgs){const t=img.imageType||'unclassified';types[t]=(types[t]||0)+1;}
const summary=Object.entries(types).map(([k,v])=>k+':'+v).join(', ');
console.log('ℹ️  Image types: '+summary);
const headshots=imgs.filter(i=>i.imageType==='headshot');
const named=headshots.filter(i=>i.associatedName);
if(headshots.length>0) console.log('ℹ️  Headshots: '+headshots.length+' total, '+named.length+' with associatedName');
" 2>/dev/null

# 11. Team member photo coverage
node -e "
const d=JSON.parse(require('fs').readFileSync('$OUTPUT_DIR/business-data.json','utf8'));
const team=d.team||[];
if(team.length===0){console.log('ℹ️  No team members extracted');process.exit(0);}
const withPhoto=team.filter(m=>m.imageUrl);
console.log('ℹ️  Team: '+team.length+' members, '+withPhoto.length+' with photo URL');
if(withPhoto.length<team.length) console.log('⚠️  Some team members missing photo URLs');
" 2>/dev/null

echo "=== End Verification ==="
```

If images are missing (⚠️ on check 3), **inform the user** explicitly:

> The scraper could not download images from the profile. The website will
> use stock photography as a fallback. For better results, you can save
> images from your profile manually to `src/data/assets/`.

If sections are sparse (⚠️ on check 6), **inform the user**:

> The scraper extracted limited content from the URL. The generated website
> may have sparse sections. For better results, you can provide additional
> details about your business (team members, services, portfolio items, etc.)
> directly in the chat.

### Step 5: Review Extracted Data

Read the generated `business-data.json` and verify:

1. **Business name** — present and correct
2. **Description** — meaningful content (not empty or boilerplate)
3. **Address / location** — present for local businesses
4. **Phone** — present if listed
5. **Hours** — present if listed
6. **Images** — at least 1 downloaded, or user has been informed they're missing
7. **Sections** — check if meaningful sections were extracted (hero, about,
   services, etc.). For generic websites, the scraper now extracts
   section-by-section content with type classification.
8. **Sub-pages** — if the site has a sitemap or navigation links, check that
   high-priority sub-pages (about, team, portfolio) were crawled
9. **Team / Portfolio** — if extracted, verify names and roles are reasonable
10. **Brand signals** — note any extracted colors/fonts for downstream skills

If critical fields are missing, ask the user to fill in the gaps before
proceeding to the build step.

### Step 6: Pass Data to Build Pipeline

The extracted data is now available for downstream skills. When building
components, reference the data file:

```tsx
import businessData from '@/data/business-data.json'
```

For images downloaded to `src/data/assets/`, reference them in components:

```tsx
<img src={new URL('./data/assets/photo-0.jpg', import.meta.url).href} alt="..." />
```

Or if using Vite, import directly:

```tsx
import heroImage from '@/data/assets/photo-0.jpg'
```

---

## Output Schema

The scraper produces JSON conforming to this shape (backward-compatible —
all new fields are optional):

```typescript
interface SiteData {
  // ── Core business info (same as before) ──
  name: string
  description: string
  address?: string
  city?: string
  state?: string
  zip?: string
  country?: string
  phone?: string
  website?: string
  rating?: number
  reviewCount?: number
  priceLevel?: string
  categories?: string[]
  hours?: Record<string, string>
  images: {
    url: string
    localPath?: string
    alt?: string
    imageType?: 'og-image' | 'logo' | 'banner' | 'headshot'
               | 'product-logo' | 'scene' | 'icon' | 'unknown'
    sectionType?: string    // which section the image belongs to
    associatedName?: string // person/company name the image is associated with
    width?: number
    height?: number
  }[]
  reviews?: {
    author: string
    rating: number
    text: string
    date?: string
  }[]

  // ── Site structure (new — generic websites) ──
  navigation?: {             // extracted from <nav> / <header>
    label: string
    href: string
    children?: NavItem[]
  }[]
  sections?: {               // page sections with type classification
    tag: string              // e.g. "section", "article", "div"
    heading?: string
    text?: string            // first ~500 chars of section body
    imageUrls: string[]
    links: string[]
    sectionType?: string     // classified: hero|about|team|portfolio|services|
                             // features|pricing|testimonials|blog|contact|
                             // faq|cta|stats|partners|process|unknown
  }[]
  team?: {                   // extracted from team/people sections
    name: string
    role?: string
    bio?: string
    imageUrl?: string
  }[]
  portfolio?: {              // extracted from portfolio/projects/investments
    name: string
    description?: string
    imageUrl?: string
    url?: string
    category?: string
  }[]

  // ── Brand signals (new) ──
  brandColors?: string[]     // CSS custom properties or computed colors
  fonts?: string[]           // Google Fonts or computed font families

  // ── Sub-page data (new — sitemap/nav-based crawling) ──
  subPages?: {
    url: string
    title: string
    path: string             // e.g. "/team", "/portfolio"
    sections: PageSection[]
    images: SiteImage[]
  }[]
  sitemapPageCount?: number  // total pages found in sitemap.xml

  // ── Metadata ──
  sourceUrl: string
  sourceType: string
  scrapedAt: string
  accessLevel: 'full' | 'partial' | 'restricted'
}
```

### Image Types Reference

Each image in `images[]` now carries an `imageType` classification based on
its context on the page:

| Type | Meaning | How detected |
|---|---|---|
| `og-image` | OpenGraph meta image (site logo / social preview) | `<meta property="og:image">` |
| `logo` | Brand logo / icon | In `<header>/<nav>`, alt text contains "logo", or URL contains "logo" |
| `banner` | Wide promotional image / hero banner | Aspect ratio > 2.5:1 and width > 400px |
| `headshot` | Person portrait photo (team member) | Inside a team-classified section, associated with a name |
| `product-logo` | Company/product logo in portfolio | Small image inside a portfolio section |
| `scene` | General scene / large photo | Large image not fitting other categories |
| `icon` | Small icon or decorative element | Width < 50px (filtered out) |
| `unknown` | Could not determine type | Default when no rule matches |

**Downstream usage:** Components MUST use `imageType` and `associatedName`
to map images to sections, NOT sequential indexing (photo-0, photo-1...).
Team sections must only use images where `imageType === 'headshot'` and
match `associatedName` to the team member's name. Hero sections must NOT
use `og-image` type images as backgrounds (they are typically small logos).

### Section Types Reference

The scraper classifies extracted sections into these types based on
element class/id names and heading text (supports both English and Chinese
keywords):

| Type | Matches | Typical content |
|---|---|---|
| `hero` | hero, banner, masthead, 首屏, 主视觉 | Main headline + CTA + visual |
| `about` | about, story, mission, philosophy, 关于, 简介, 介绍 | Brand narrative, values |
| `team` | team, people, leadership, 团队, 成员, 管理层 | Member cards with name/role/photo |
| `portfolio` | portfolio, investments, projects, gallery, 投资, 项目, 案例 | Work items / case studies |
| `services` | service, solution, capabilities, 服务, 解决方案, 业务 | Service descriptions |
| `features` | feature, benefit, why-us, 特色, 优势, 亮点 | Feature grid / highlights |
| `pricing` | pricing, plans, packages, 价格, 套餐 | Pricing tiers |
| `testimonials` | testimonial, review, feedback, 评价, 客户, 反馈 | Client quotes |
| `blog` | blog, insights, news, resources, 新闻, 动态, 资讯 | Article cards |
| `contact` | contact, get-in-touch, location, 联系, 地址, 位置 | Contact form / map |
| `faq` | faq, questions, help, 常见问题, 帮助 | Q&A accordion |
| `cta` | cta, get-started, sign-up | Call-to-action band |
| `stats` | stats, numbers, achievements, 数据, 成就, 指数 | Key metrics / counters |
| `partners` | partner, logos, trusted, 合作, 伙伴 | Client/partner logo grid |
| `process` | process, how-it-works, steps, 流程, 步骤 | Step-by-step workflow |
| `unknown` | (no match) | Unclassified section |

---

## Fallback Behavior

If the scraper fails entirely (network error, page structure changed, etc.):

1. **Log the error clearly** — show the user the exact error message
2. **Inform the user what happened** — explain which data is missing and why
3. **Ask the user to provide business details manually** — name, description,
   address, phone, hours, and any images they can supply
4. Continue to the build pipeline with whatever data is available

**What you must NOT do on failure:**
- ❌ Silently substitute web search results for the scraper output
- ❌ Hand-write `business-data.json` using data from third-party websites
- ❌ Use Unsplash/Pexels images without telling the user the scraper failed
- ❌ Pretend the scraper ran when it didn't
- ❌ Skip the scraper preemptively because a simple `WebFetch` failed —
  `WebFetch` is not the scraper; the Playwright-based scraper handles
  JavaScript-rendered pages that `WebFetch` cannot

Never block the entire website generation because scraping failed — but
always be transparent with the user about what data is real vs. substituted.
