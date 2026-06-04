---
name: migration-score
description: Computes a structured AEM-to-EDS migration assessment from a block inventory. Takes block counts by complexity tier, page volume, and optional risk modifiers; produces a Migration Score (0–100, higher = easier migration), ease rating (Easy / Moderate / Hard / Very Hard), phased timeline estimate, and the set of adjustment factors applied. Designed to be called from orchestrator skills such as customer-pov and analyze-and-plan.
license: Apache-2.0
metadata:
  version: "1.0.0"
---

# Migration Score — AEM to EDS

Compute a structured migration assessment from a block inventory and site metrics. Returns a migration score, complexity rating, phased timeline estimate, and applied adjustments.

---

## When to Use This Skill

Use this skill when:
- You have a completed block inventory (counts by type) and a page count from sitemap analysis
- You need a repeatable, documented migration complexity rating for a customer POV or migration plan
- An orchestrator skill (customer-pov, analyze-and-plan) needs to populate a migration scope section

**Do NOT use this skill when:**
- You have not yet run block-inventory or page-decomposition — gather those inputs first
- You only need a rough verbal estimate with no structured output

## Why This Skill Exists

Migration complexity is computed the same way across multiple orchestrator skills. This skill defines the velocity table, adjustment factors, and scoring formula in one place so they are not duplicated across customer-pov, analyze-and-plan, and any future skill that needs migration estimates.

## Related Skills

- **customer-pov** — calls this skill after Step 2e/Step 5 to populate the Migration Scope Estimate section
- **analyze-and-plan** — can call this skill when planning a migration project scope
- **block-inventory** — provides the block counts this skill needs as input
- **page-decomposition** — provides service-endpoint component data this skill needs as input

---

## Input Contract

Before running this skill, the caller must supply the following data. All counts are integers. Modifiers are booleans unless stated otherwise.

### Block Inventory (required)

| Input | Description | Default if unknown |
|---|---|---|
| `blocks_adopt` | Blocks covered by EDS Block Collection, usable as-is | 0 |
| `blocks_customize` | Blocks needing minor customization of an existing collection block | 0 |
| `blocks_custom` | Net-new custom blocks with no collection equivalent | 0 |
| `blocks_service_simple` | Service-endpoint blocks using static fetch / query-index pattern | 0 |
| `blocks_service_complex` | Service-endpoint blocks requiring auth, edge proxy, or real-time data | 0 |
| `blocks_spa` | SPA sections requiring full re-implementation in plain JS | 0 |

### Site Metrics (required)

| Input | Description |
|---|---|
| `total_pages` | Total page count from sitemap analysis |
| `template_count` | Number of distinct page template types |

### Risk Modifiers (optional — default false / 0 if not provided)

| Input | Description |
|---|---|
| `locale_count` | Number of languages / locales (integer; triggers +25% if ≥ 3) |
| `has_auth_personalization` | True if auth-gated or personalized content spans many page types |
| `has_formal_qa` | True if customer has a formal, gated UAT / QA process |
| `already_uses_docs` | True if customer already authors in Google Docs or SharePoint today |
| `dominant_template` | True if ≥ 50% of pages share a single template |

---

## Step 1 — Classify Block Complexity

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

## Step 2 — Apply Reference Velocities

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

## Step 3 — Apply Adjustment Factors

Apply each matching modifier to `raw_effort`. Record every factor applied — it will appear in the output assumptions list.

| Modifier | Condition | Adjustment |
|---|---|---|
| Multi-locale overhead | `locale_count` ≥ 3 | +25% |
| Auth / personalization complexity | `has_auth_personalization` is true | +20% |
| Formal QA / UAT gating | `has_formal_qa` is true | +15% |
| Already uses Docs / SharePoint | `already_uses_docs` is true | −15% |
| Dominant single template | `dominant_template` is true | −10% |

Apply all matching adjustments multiplicatively:

```
adjusted_effort = raw_effort × (1 + adj_1) × (1 + adj_2) × ...
```

If no modifiers apply, `adjusted_effort = raw_effort`.

---

## Step 4 — Compute Migration Score (0–100)

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
| `already_uses_docs` | −5 |

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

## Step 5 — Build Phase Timeline

Divide `adjusted_effort` across phases using these allocation rules. Derive concrete week / quarter estimates from the block and page effort numbers computed in Steps 2–3.

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

## Step 6 — Return Structured Output

Produce the following structured result. The calling skill embeds this output in its own document.

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

[Insert completed phase timeline table from Step 5]

---

### Assumptions & Adjustments Applied

[List every adjustment applied, one bullet per line:]
• +25% — 3+ locales detected (translation coordination overhead)
• +20% — auth / personalization present across page types
• −15% — customer already authors in SharePoint / Google Docs

[If no adjustments applied, write:]
• No adjustments applied — baseline estimate used.
```
