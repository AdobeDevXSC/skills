---
name: migration-score
description: Computes a structured AEM-to-EDS migration assessment by automatically analyzing a customer URL. Fetches the sitemap to measure page volume and locales, scrapes the homepage and sample pages to classify block complexity, then produces a Migration Score (0–100, higher = easier migration), ease rating (Easy / Moderate / Hard / Very Hard), phased timeline estimate, and the set of adjustment factors applied.
license: Apache-2.0
metadata:
  version: "2.0.0"
---

# Migration Score — AEM to EDS

Automatically analyze a customer site and compute a structured migration assessment. Requires only the customer URL — all block inventory, page metrics, and risk modifiers are discovered from the live site.

---

## When to Use This Skill

Use this skill when:
- You have a customer URL and need a migration complexity rating
- You need a repeatable, documented migration complexity rating for a customer POV or migration plan
- An orchestrator skill (analyze-and-plan) needs to populate a migration scope section

**Do NOT use this skill when:**
- You only need a rough verbal estimate with no structured output

## Why This Skill Exists

Migration complexity is computed the same way across multiple orchestrator skills. This skill defines the velocity table, adjustment factors, and scoring formula in one place so they are not duplicated across analyze-and-plan and any future skill that needs migration estimates. Auto-discovery removes the need for manual block counting.

## Related Skills

- **analyze-and-plan** — can call this skill when planning a migration project scope
- **scrape-webpage** — used by this skill to fetch and analyze sample pages
- **page-decomposition** — provides additional section-level analysis if needed

---

## Step 0 — Prompt for Customer URL

If `customer_url` has not been provided by the caller or the user, ask before proceeding:

> **What is the customer's website URL?**
> Please provide the root URL of the AEM site to assess (e.g., `https://www.example.com`).

Do not proceed until a URL is supplied. Extract the hostname (e.g., `example.com`) — use it as the default customer name slug for the output filename.

---

## Step 1 — Fetch Sitemap and Measure Site Scale

Fetch the sitemap to count pages, detect locales, and identify template patterns. Try these URLs in order, stopping at the first that returns valid XML:

1. `{customer_url}/sitemap.xml`
2. `{customer_url}/sitemap_index.xml`
3. `{customer_url}/robots.txt` → extract `Sitemap:` directive, then fetch that URL

Use the **WebFetch** tool for each request.

**From the sitemap XML, extract:**

- **`total_pages`** — count all `<loc>` entries across all sitemaps (for sitemap index files, sum across child sitemaps; cap at 5 child sitemaps to avoid excessive fetches)
- **`locale_count`** — count distinct locale path segments in URLs. Look for patterns like `/en/`, `/de/`, `/fr/`, `/ja/`, `/es/`, `/pt/`, `/ko/`, `/zh/`, etc. Each distinct two-letter or IETF language tag counts as one locale. If no locale segments found, set `locale_count = 1`.
- **`template_count`** — group URLs by path depth and leading path segment (e.g., `/products/`, `/blog/`, `/solutions/`). Count distinct top-level path groups as distinct templates.
- **`dominant_template`** — set `true` if one path group accounts for ≥ 50% of all URLs.
- **Sample URLs** — collect 8–12 representative URLs spread across different path groups for Step 2. Prefer URLs at path depth 2–3 (not just the homepage).

If the sitemap is unavailable or returns an error, set `total_pages = 100` (conservative default), note the assumption, and proceed.

---

## Step 2 — Scrape Homepage and Sample Pages

Use the **WebFetch** tool to fetch the raw HTML of the homepage and 3–5 sample URLs collected in Step 1 (aim for variety across path groups). Fetch them in parallel.

For each page fetched, extract:
- All `<script>` tag `src` attributes and inline content
- All HTML structural elements: `<section>`, `<article>`, `<aside>`, `<div>` class names
- Any `data-*` attributes that suggest component or block systems
- Forms, authentication markers, and personalization tokens
- Meta tags: `<meta name="generator">`, framework fingerprints

Store the combined HTML signal set for classification in Step 3.

---

## Step 3 — Classify Block Inventory

Analyze the scraped HTML signals from Step 2 to classify blocks into the six migration tiers. Apply each rule set to all pages and sum totals.

### SPA Detection → `blocks_spa`

Count each distinct SPA-rendered section as one `blocks_spa` unit. Signals:

| Signal | Indicator |
|---|---|
| `<div id="root">` or `<div id="app">` with minimal server-rendered content | React / Vue SPA |
| `__NEXT_DATA__` script tag or `/_next/` script paths | Next.js |
| `ng-version` attribute or `angular.json` references | Angular |
| `window.__nuxt__` or `_nuxt/` script paths | Nuxt.js |
| `data-reactroot` or `data-reactid` attributes | React |
| Script bundles > 500 KB loaded for primary content | Heavy JS rendering |

For each distinct SPA-rendered page section or route detected across sample pages, add 1 to `blocks_spa`. Cap at 10 (larger SPAs are still counted as one workstream).

### Complex Service Blocks → `blocks_service_complex`

Signals indicating auth-gated or real-time data endpoints:

| Signal | Indicator |
|---|---|
| Login / sign-in forms or links in navigation | Auth-gated content |
| References to session tokens, OAuth, SSO | Authentication layer |
| Personalization tokens: `{firstName}`, `{{user}}`, Adobe Target `mbox` | Personalization |
| Real-time pricing, inventory, or stock data | Live data feeds |
| Chat widgets tied to account data | Complex service integration |

Count distinct service integration patterns found. Set `has_auth_personalization = true` if any auth or personalization signal is present.

### Simple Service Blocks → `blocks_service_simple`

Signals indicating static API or query-index patterns:

| Signal | Indicator |
|---|---|
| `query-index.json` or `/index.json` fetch patterns | EDS query index |
| Static JSON data files loaded via fetch | Static data feed |
| Simple search widgets without auth | Query-based content |
| Blog / news listing components pulling from a feed | Feed aggregation |

Count distinct static fetch patterns found across sample pages.

### Net-new Custom Blocks → `blocks_custom`

Signals for unique interactive components with no EDS Block Collection equivalent:

| Signal | Indicator |
|---|---|
| Complex interactive calculators or configurators | Custom logic required |
| Multi-step forms with branching logic | Custom block |
| Video players with custom controls beyond standard `<video>` | Custom media block |
| Data visualization (charts, maps, dashboards) | Custom block |
| E-commerce cart, checkout, or product configurator | Custom block |
| Custom navigation mega-menus with deep nesting | Custom block |

Count distinct custom component types found.

### Customizable Collection Blocks → `blocks_customize`

Signals for components that resemble EDS Block Collection blocks but with modifications:

| Signal | Indicator |
|---|---|
| Hero sections with non-standard layouts (video background, split layout) | Customize hero |
| Card grids with extra fields beyond image/title/text | Customize cards |
| Tabs or accordions with styling variations | Customize tabs/accordion |
| Carousels with custom controls or auto-play behavior | Customize carousel |
| Navigation patterns close to standard but with brand additions | Customize nav |

Count distinct component types in this category.

### Adopt-as-is Collection Blocks → `blocks_adopt`

Signals for components directly covered by EDS Block Collection:

| Signal | Indicator |
|---|---|
| Standard hero sections (heading, paragraph, buttons) | Adopt hero |
| Simple image + text card grids | Adopt cards |
| Two/three-column text layouts | Adopt columns |
| Standard FAQ accordion | Adopt accordion |
| Basic tab navigation | Adopt tabs |
| Simple image carousel | Adopt carousel |
| Pull-quote or testimonial sections | Adopt quote |
| Reusable content fragments | Adopt fragment |

Count distinct standard block types found.

---

## Step 4 — Finalize Risk Modifiers

Combine signals from Steps 1–3 to set each risk modifier:

| Modifier | Source | Value |
|---|---|---|
| `locale_count` | Sitemap URL patterns (Step 1) | Integer count |
| `has_auth_personalization` | Auth/personalization signals (Step 3) | true / false |
| `has_formal_qa` | Cannot be auto-detected | Default: **false** — note assumption |
| `dominant_template` | Sitemap path distribution (Step 1) | true / false |

Print a discovery summary before proceeding to scoring:

```
Site Discovery Summary — {customer_url}
─────────────────────────────────────────
Total pages:          {total_pages}
Locales detected:     {locale_count}
Templates detected:   {template_count}
Dominant template:    {yes/no}

Block classification:
  Adopt as-is:        {blocks_adopt}
  Customize:          {blocks_customize}
  Custom (net-new):   {blocks_custom}
  Service (simple):   {blocks_service_simple}
  Service (complex):  {blocks_service_complex}
  SPA sections:       {blocks_spa}

Risk modifiers:
  Auth/personalization: {yes/no}
  Formal QA process:    assumed no (could not detect)
```

---

## Step 5 — Classify Block Complexity

Compute the total block count and custom block ratio:

```
total_blocks   = blocks_adopt + blocks_customize + blocks_custom
               + blocks_service_simple + blocks_service_complex + blocks_spa

complex_blocks = blocks_custom + blocks_service_complex + blocks_spa

custom_ratio   = complex_blocks / total_blocks   (0 if total_blocks = 0)
```

Classify the migration's **base complexity tier** using the first matching rule:

| Rule | Base Complexity |
|---|---|
| `blocks_spa` > 0 | **High** |
| `custom_ratio` > 0.40 OR `blocks_service_complex` > 2 | **High** |
| `custom_ratio` 0.20–0.40 OR `has_auth_personalization` is true | **Medium** |
| `total_pages` > 500 AND `custom_ratio` > 0.10 | **Medium** |
| Otherwise | **Low** |

If multiple rules match, use the highest tier.

---

## Step 6 — Apply Reference Velocities

Using the base complexity tier, look up each work item's velocity. Use the **midpoint** of each range for effort calculations.

**Reference velocities (1 developer + 1 content author pair):**

| Work Item | Low | Medium | High |
|---|---|---|---|
| Block Collection — adopt as-is | 0.5 days | 0.5 days | 0.5 days |
| Block Collection — minor customization | 1–2 days | 2–3 days | 3–5 days |
| Net-new custom block | 3–5 days | 5–8 days | 8–15 days |
| Service-endpoint block — static fetch | 1–2 days | 2–3 days | 3–5 days |
| Service-endpoint block — auth / edge proxy | 3–5 days | 5–8 days | 8–15 days |
| SPA section re-implementation | 10–20 days | 20–40 days | 40+ days |
| Page content migration (bulk, author-driven) | 50–100 pages/day | 20–50 pages/day | 10–20 pages/day |
| CI/CD pipeline + GitHub repo setup | 2–3 days | 3–5 days | 5–8 days |
| Author onboarding & training | 1–2 days | 2–3 days | 3–5 days |

Compute **block build effort** (developer-days):

```
block_effort =
    (blocks_adopt           × 0.5)
  + (blocks_customize       × midpoint(customization_range))
  + (blocks_custom          × midpoint(custom_block_range))
  + (blocks_service_simple  × midpoint(service_simple_range))
  + (blocks_service_complex × midpoint(service_complex_range))
  + (blocks_spa             × midpoint(spa_range))
  + midpoint(cicd_range)
  + midpoint(training_range)
```

Compute **page migration effort** (developer-days):

```
page_effort = total_pages / midpoint(page_velocity_range)
```

Compute **total raw effort**:

```
raw_effort = block_effort + page_effort
```

---

## Step 7 — Apply Adjustment Factors

Apply each matching modifier to `raw_effort`. Record every factor applied — it will appear in the output assumptions list.

| Modifier | Condition | Adjustment |
|---|---|---|
| Multi-locale overhead | `locale_count` ≥ 3 | +25% |
| Auth / personalization complexity | `has_auth_personalization` is true | +20% |
| Formal QA / UAT gating | `has_formal_qa` is true | +15% |
| Dominant single template | `dominant_template` is true | −10% |

Apply all matching adjustments multiplicatively:

```
adjusted_effort = raw_effort × (1 + adj_1) × (1 + adj_2) × ...
```

If no modifiers apply, `adjusted_effort = raw_effort`.

---

## Step 8 — Compute Migration Score (0–100)

The Migration Score is a normalized composite index — a single comparable number for stakeholder summaries. **Higher score = easier migration; lower score = more complex migration.** It supplements the timeline estimate; it does not replace it.

Compute complexity penalty sub-scores, then subtract from 100.

**Block complexity penalty (0–60):**

| Condition | Sub-score |
|---|---|
| `custom_ratio` = 0 AND no SPA blocks | 0 |
| `custom_ratio` > 0 and ≤ 0.20, no SPA | 10 |
| `custom_ratio` > 0.20 and ≤ 0.40, no SPA | 25 |
| `custom_ratio` > 0.40, no SPA | 40 |
| Any ratio, 1–2 SPA blocks present | 50 |
| Any ratio, 3+ SPA blocks present | 60 |

**Page volume penalty (0–25):**

| `total_pages` | Sub-score |
|---|---|
| < 50 | 5 |
| 50–200 | 10 |
| 201–500 | 15 |
| 501–2000 | 20 |
| > 2000 | 25 |

**Risk modifier penalty (−5 to +15):**

| Modifier | Points |
|---|---|
| `locale_count` ≥ 3 | +5 |
| `has_auth_personalization` | +5 |
| `has_formal_qa` | +5 |

```
complexity_penalty = clamp(block_sub + page_sub + risk_sub, 0, 100)
migration_score    = 100 − complexity_penalty
```

Map score to ease label:

| Score | Label |
|---|---|
| 76–100 | **Easy** |
| 51–75 | **Moderate** |
| 26–50 | **Hard** |
| 0–25 | **Very Hard** |

---

## Step 9 — Build Phase Timeline

Divide `adjusted_effort` across phases using these allocation rules. Derive concrete week / quarter estimates from the block and page effort numbers computed in Steps 6–7.

**Phase 0 — POC (target: 2–4 weeks)**
- Scope: 1 representative page (homepage or highest-traffic landing page); typically 4–8 blocks
- Effort: sum of block estimates for those blocks at the applicable velocity + CI/CD setup
- If effort exceeds 4 weeks, recommend narrowing page scope or selecting a simpler page

**Phase 1 — Pilot (target: 6–12 weeks)**
- Scope: 1 site section or new campaign site; typically 10–30 pages, 10–20 blocks
- Effort: block builds not completed in Phase 0 + page migration for pilot pages + author training

**Phase 2 — Scaled Migration (ongoing, per quarter)**
- Scope: remaining pages by template type
- Effort: remaining block builds + bulk page migration
- Round up to the nearest full quarter
- Flag any SPA sections or auth-gated components as separate workstreams with their own timelines

**Phase timeline output table:**

| Phase | Scope Summary | Estimated Duration | Key Dependencies |
|---|---|---|---|
| Phase 0 — POC | 1 page, ~N blocks | N weeks | Block Collection availability, CDN / GitHub setup |
| Phase 1 — Pilot | N pages, N new blocks | N weeks | Author training, CI/CD pipeline |
| Phase 2 — Scaled Migration | N pages across N templates | N quarters | Content freeze windows, locale sign-off |
| SPA / Complex Components _(if applicable)_ | List components | Separate workstream | Architecture decision required |
| **Total (Phase 0 → full production)** | | **~N months** | |

---

## Step 10 — Save and Return Structured Output

Produce the following structured result. When called standalone, save it to the output subfolder (instructions below). When called by an orchestrator skill, the calling skill is responsible for embedding or persisting this output.

### Output File

1. **Resolve the output directory** — run this to get the absolute path:
   ```bash
   SKILL_DIR="$(git rev-parse --show-toplevel)/plugins/aem/edge-delivery-services/skills/migration-score"
   mkdir -p "$SKILL_DIR/output"
   echo "$SKILL_DIR/output"
   ```
   Use the printed path in the next step.

2. **Derive the filename:**
   - If a customer or project name was provided, use it as a slug: lowercase, spaces and special characters replaced with hyphens.
   - If no explicit name was provided, derive the slug from the `customer_url` hostname (e.g., `https://www.example.com` → `example-com`).
   - Format: `<slug>-migration-score-<YYYY-MM-DD>.md` (e.g., `acme-corp-migration-score-2026-06-07.md`)
   - If neither a name nor a URL was provided, use: `migration-score-<YYYY-MM-DD>.md`

3. **Write the file** using the Write tool to `<output_dir>/<filename>`.

4. **Confirm the saved path** to the user:
   > Migration score saved to `plugins/aem/edge-delivery-services/skills/migration-score/output/<filename>`

### Structured Output

```markdown
### Migration Score

**Score:** NN / 100 — <Label> _(higher = easier migration)_

**Complexity Rating:** Low / Medium / High / Very High

**Migration Ease:** Easy / Moderate / Hard / Very Hard

**Adjusted Effort Estimate:** ~NN developer-days

---

### Block Inventory Summary

| Metric | Count |
|---|---|
| Blocks — adopt as-is | |
| Blocks — minor customization | |
| Net-new custom blocks | |
| Service-endpoint blocks (simple fetch) | |
| Service-endpoint blocks (auth / complex) | |
| SPA sections | |
| **Total unique block types** | |
| **Complex block ratio** | NN% |

---

### Phase Timeline

[Insert completed phase timeline table from Step 9]

---

### Assumptions & Adjustments Applied

[List every adjustment applied, one bullet per line:]
• +25% — 3+ locales detected (translation coordination overhead)
• +20% — auth / personalization present across page types

[Note any values that could not be auto-detected and were defaulted:]
• has_formal_qa — could not be detected from public site; assumed false

[If no adjustments applied, write:]
• No adjustments applied — baseline estimate used.
```

---

## Step 11 — Deliver and Offer Next Actions

After presenting the migration score output, offer the following options:

1. **Refine the inputs** — "Would you like to adjust any block counts, page volumes, or risk modifiers and recompute the score?"
2. **Drill into a phase** — "Would you like a more detailed breakdown of Phase 0 or Phase 1 scope and effort?"
3. **Generate a PPTX presentation** — "Would you like me to produce a PowerPoint presentation of this migration assessment? I'll use the official Adobe EDS POV slide templates and populate them with the migration score, timeline, and assumptions." _(Invoke the `/generate-pptx` skill, passing the customer/project name and the migration score output generated above.)_
4. **Feed into a full POV** — "Would you like to incorporate this migration score into a full customer POV? I'll carry all of the migration score output into `/customer-pov` so the migration approach section is pre-populated with the actual score, block inventory, phase timeline, and assumptions — no re-entry needed."

   When the user selects this option, invoke the `/customer-pov` skill and pass the **complete migration score output** as a named context block so customer-pov can use it verbatim. Specifically:
   - Pass the customer URL (or customer name slug) so customer-pov does not need to re-ask.
   - Pass the full structured output produced in Step 10 — Migration Score, Block Inventory Summary, Phase Timeline table, and Assumptions & Adjustments Applied — as `migration_score_output`.
   - Tell customer-pov: **"Use the provided `migration_score_output` to populate the Recommended Migration Approach section. Do not generate new phase estimates — use the Phase Timeline table from the migration score exactly as produced. Embed the Migration Score, ease rating, adjusted effort, and block inventory summary in that section as well."**

After any refinement accepted by the user, overwrite the existing output file with the updated content using the Write tool (same path as Step 10). Confirm the update to the user.
