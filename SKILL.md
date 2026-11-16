---
name: supplier-performance-review
description: >
  Analyse supplier performance against contracted SLAs in the Anaplan PRM model —
  On-Time Delivery %, Delivery Fill Rate %, Lead Time, Supplier Tier, To Review flags,
  Total Spend and AOV — and produce a ranked underperformer action list for a
  Procurement or Sourcing Manager. Use this skill whenever the user mentions suppliers,
  vendors, SLAs, OTD, DFR, fill rate, lead time, supplier tiers, supplier scorecards,
  spend at risk, late deliveries, short shipments, "who should I chase", "which
  suppliers are underperforming", supplier reviews, QBRs, dual-sourcing decisions, or
  asks about performance at a named DC, plant, location or region — even if they never
  say the words "supplier performance" or name a module. Also use it for a narrower
  follow-up about one supplier, one product or one site: the same cached coordinates
  and the same data traps apply.
compatibility: >
  Anaplan MCP tool set, bound to the PRM supplier performance model.
  Fast path = two tools: get_model_context, read_cells.
---

# Supplier Performance Review

## Speed rules (read first)

1. **One `read_cells` call.** Copy the literals in Step 1 exactly. Do not call
   `catalog_modules` or `catalog_line_items` to "check" them — that check is what this
   skill replaces. A failed call costs the round trip, a diagnostic call, and a retry.
2. **Answer in ≤400 words.** The manager asked who to chase. Answer that, then stop.
3. **No artifacts unless asked.** See *Output discipline* — this is the single biggest
   driver of response time.

---

## Step 0 — Confirm binding

```
get_model_context()
```

Bound to `PRM - PR Management 1.0`? Go to Step 1. **Do not rebind** — it burns a
model-selection slot and clears the time anchor.

Only if nothing is bound:

```
set_model_context(
  workspace_id = "3deff6878f494b89b0a171500228b68d",
  model_id     = "F23AD4B240BC44709046726A36FB6F9F"
)
```

If several PRM variants exist, show the list and let the user pick — tier thresholds and
SLA baselines differ between copies, so guessing produces a confident wrong answer.

## Step 1 — The one read

Module **`KPI220 - Supplier Performance Reporting`** (id `102000000594`). Three
dimensions, no others:

| Dimension (exact string) | Use |
|---|---|
| `HLS080 - Supplier Leaf #` | leaf |
| `HLP080 - Product Leaf #` | leaf |
| `HLL080 - Location Leaf #` | slice to the user's site |

```
read_cells(
  module = "KPI220 - Supplier Performance Reporting",
  line_items = [
    "Final OTD%", "Final DFR%", "Final Lead Time",
    "OTD Variance", "DFR Variance", "Lead Time Variance",
    "Final Supplier Tier", "To Review?",
    "Total Spend", "AOV"
  ],
  slice = { "HLL080 - Location Leaf #": "Florida DC" },
  leaf_dimensions = ["HLS080 - Supplier Leaf #", "HLP080 - Product Leaf #"],
  limit = 99
)
```

One DC returns the complete picture (~42 rows at Florida) in ~10 seconds.

**`limit = 99` — not 100, not 200.** Exclusive-read SQL caps at 100 rows and the tool
adds one internally to test for a next page. Anything above 99 fails with
`CORE_API_HTTP_ERROR 400` before touching data.

**Spellings.** No space before `%`: `Final OTD%`, not `Final OTD %`. One wrong character
fails the call. Baselines are `Contract OTD%` / `Contract DFR%` / `Contract Lead Time` —
the model never uses the word "SLA". They are omitted above on purpose: the variances
already encode actual-minus-contract. Add only if the user asks what was contracted.

Optional, on demand only: `Priority (Compound Alerts)`, `Prior OTD`, `Prior DFR`,
`Prior Supplier Tier`, `Change in Supplier Tier?`, `AOV as % of Spend`, `PY Total Spend`,
`Override OTD%`.

**If it fails** — stop at the first fix, never escalate to `sql_schema` exploration:
row-cap error → `limit = 99`. Unknown line item → drop the optional items, retry.
Unaccounted dimension → the error names it, add to `slice`, retry. Still failing after
two attempts → `catalog_line_items(module_id="102000000594", include_dimensions=false,
limit=100)` and tell the user the cached coordinates need updating.

## Step 2 — Scope

Location named? Use it. Not named? **Do not ask** — run their known site or the top-level
roll-up, state the scope in line one, offer to narrow. A clarifying question costs a full
round trip; a stated assumption costs one sentence. Interrupt only for genuine ambiguity
(a name matching both a region and a DC inside it).

---

## Reading values correctly

| Item | Stored as | Means |
|---|---|---|
| `Final OTD%`, `Final DFR%` | ratio — `0.625` | 62.5% — ×100 |
| `OTD Variance`, `DFR Variance` | **percentage points** — `-17.5` | 17.5 pts below contract |
| `Lead Time Variance`, `Final Lead Time` | days — `-0.75` | 0.75 days faster |

Different units in the same read. Never ×100 a variance. Long floats
(`-30.000000000000004`) are artefacts of ±30 — round for display, treat within 1e-9 of
zero as exactly zero, because that zero test separates real data from missing data.

## Traps

1. **Ratios don't roll up.** `Final OTD%`, `Final DFR%` and all variances are blank at
   `All Products` / `All Suppliers`. Blank means "doesn't aggregate", not "failing".
   `Total Spend` does roll up — use it to validate leaf sums. **Analyse at
   Supplier × Product leaf.**
2. **`0%/0%` with zero variance is missing data, not zero performance.** No delivery
   records flowing. These are **not** flagged To Review, so they're invisible — and some
   carry real spend. Report as "blind spots" with spend attached; **never rank as
   underperformers**. Telling a manager to chase a supplier actually running at 100%
   destroys trust in the report. Zero variance *and* zero spend = unused pair, not a
   finding.
3. **`To Review?` false-positives.** Fires on *category change*, including improvement.
   Expect flagged lines beating SLA on all three metrics. Name them, recommend
   `Clear Review`.
4. **Negative Lead Time variance is good** — faster than contracted. Never a breach. LT
   breaches are rare here; **if there are zero, say so prominently** — it moves the
   supplier conversation off planning horizons onto reliability and capacity.
5. **No Time, no Versions dimension.** Horizon is set in
   `KPI100 - Supplier KPI Time Settings`. Never filter by period or ask about versions.
6. **Tier is supplier-level**, repeating across every row. A Gold supplier can carry a
   50% line; a "Needs Improvement" supplier can be near-perfect bar one SKU. Never infer
   line health from tier.

## Analysis recipe

Free — the data is already in context. Run all, report only what fires.

1. **Spend at risk per supplier** — sum `Total Spend` where OTD variance < 0. Absolute
   and % of their spend. **Rank by absolute**, not by tier or worst percentage. A 50%
   line worth $700 is not a priority; a 71% line worth $65,000 is.
2. **Breadth vs depth** — breaching lines / active lines. Four of six failing moderately
   is a systemic constraint; one of six failing severely is an isolated SKU. Different
   conversations — say which.
3. **Internal-inconsistency lever** — each underperformer's own 100% lines. Removes
   "we can't", leaves "we aren't prioritising you". Always surface.
4. **DFR-tracks-OTD** — lockstep = upstream supply constraint, don't accept a freight
   answer. OTD falls, DFR holds = scheduling/transport.
5. **Cross-supplier product scan** — if 3+ independent suppliers miss the same SKU, the
   cause is internal (spec, demand volatility, late ordering, volume spread too thin).
   **Surface above the individual chases** — one internal fix beats four phone calls.
6. **Tier-masking** — high-tier suppliers carrying material breaching spend. They never
   appear on a tier-based exception report. Shallow gaps on big spend are the highest-
   probability quick wins.
7. **Reallocation arbitrage** — same SKU, same location, one supplier at 100% and another
   failing on comparable spend. Same site = no requalification. Usually the single
   highest-value recommendation.

---

## Output discipline

**Default reply: ≤400 words, no artifacts, top 3 suppliers only.** Detail is one
follow-up away, and they'll want it for one supplier, not all six.

```markdown
# [Location] — Supplier Performance Review

**[$X at risk, N% of total spend.]** [One structural insight — often what is NOT
broken, because that reframes the conversation.]

**Chase first**
1. **[Supplier]** — $X at risk (N%). [Worst lines with OTD%/spend. The leverage
   point in one clause.]
2–3. [same, one or two lines each]

**Worth knowing** — [tier-masked supplier, or cross-supplier SKU pattern. Max two.]

**Gaps** — [blind-spot spend; false-positive flags to clear. One line each.]
```

Close with **one** offer: the full scorecard, or the same analysis at another site.

**Produce artifacts only when asked** — or when the user explicitly wants a scorecard,
export, chart, deck or something to share. Then:

- **Scorecard artifact** — every line with every metric, per-supplier talk tracks,
  actions split into this week / this month / structural, and the method note below.
- **Chart artifact** — spend at risk by supplier, breaching vs compliant vs unmeasured.

**Talk tracks** — only in the artifact, or when the user asks how to open the call. The
pattern: state their own best performance, state their worst, name the gap, name the
consequence.

> "You're at 100% on our two smallest lines. On Nurofen Tablets 12 — $171,000 — you're
> at 62.5%, and on Dettol Antiseptic — $101,000 — you're at 50%. Your fill rate tracks
> your on-time rate almost exactly, so this is an allocation decision, not a transport
> problem. We need a written recovery plan in 30 days."

**Method note** (artifact only): source module, grain and location slice; variances read
as stored, not recomputed; spend-at-risk sums are the only external arithmetic;
ratio-roll-up limitation and `Total Spend` validated against roll-up; no Time/Versions —
horizon set in `KPI100`.

## Follow-ups

- **"More on [supplier]"** / **"by product instead"** — answer from context. **No tool call.**
- **"Other DCs?"** — one `read_cells`, different slice. Multi-site can exceed 99 rows:
  one site per call, or page with `skip`.
- **"Contracted target?"** — one read adding the three `Contract` items.
- **"Who changed this?"** — `explain_cell` first (calculated cells have no history of
  their own), then `cell_change_history` with the slice copied verbatim.

A five-question session should cost two or three tool calls, not fifteen.
