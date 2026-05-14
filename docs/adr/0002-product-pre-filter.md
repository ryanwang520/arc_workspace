# Opt-in product pre-filter for large catalogs

## Context

`arc_agent`'s `search` node currently calls `fetch_all_candidates(org_id)` and ships the org's full product catalog into every BOM-generation run. This works for small and mid-sized catalogs but breaks down for orgs with 5k–20k SKUs: prompt cost balloons, latency rises, and the select-LLM's attention is diluted by irrelevant SKUs.

We previously ran an embedding-based RAG pipeline that did per-query retrieval. It was retired (see CLAUDE.md Invariants and `arc_agent/docs/eval_plan.md`) because top-K truncation was dropping relevant cold-tail products — the failure mode was **recall**, not retrieval cost.

Re-introducing a pre-filter is a deliberate revisit of that retired path, but using a structurally different approach: closed-domain set classification on every product (`material` × `product_type`), filtered by **set intersection** of customer requirements rather than top-K similarity. Recall is reasoned about per-class rather than trusted to a black-box `K`.

The `select_product/` prototype demonstrated 100% recall over 23 real customer orders against ~21k products with this shape. We borrow the architecture but re-design the schema and process to fit `arc_agent`'s constraints.

## Decision

Add an **opt-in pre-filter** that runs in the agent graph for orgs with `pre_filter_enabled = true`. Definitions of `material`, `product_type`, **applicability semantics**, and the value domains live in workspace-root `CONTEXT.md` and are canonical.

### Schema (apps/api alembic migration on shared `Base`)

```
products
  + material       TEXT[]        NOT NULL DEFAULT '{}'   -- ⊆ {wood, chain_link, vinyl, ornamental}
  + product_type   VARCHAR(32)   NULL                    -- ∈ 9-value enum (CONTEXT.md)
  + fence_height   NUMERIC(4,1)  NULL                    -- feet (e.g. 6, 6.5); matches geometry's unit
  + post_length    NUMERIC(4,1)  NULL                    -- feet
  + color          VARCHAR(32)   NULL                    -- lowercase normalized
  + classified_at  TIMESTAMPTZ   NULL
  + GIN index on material; B-tree on (org_id, product_type)
  (embedding column retained nullable per its existing ADR commitment)

org_profiles
  + pre_filter_enabled  BOOLEAN  NOT NULL DEFAULT false
```

Value domains are validated at the service layer, not via PostgreSQL ENUM types — extending the value set must not require an alembic migration.

### Onboarding (classification)

New service `apps/api/src/api/services/product_classification.py` mirrors `product_backfill.py`'s structure: async, batched (50 products / batch), Pydantic structured output, single LLM call per batch with concurrency 5. Fence-domain knowledge (Postmaster ↔ wood post; opaque short codes → wildcard; ornamental ↔ vinyl mutual exclusion; etc.) is encoded as DOMAIN HINTS **inside the prompt** — code does not carry post-hoc cleanup rules other than fuzzy-match typo correction.

Triggered async (no `upsert_product_library` hook): a one-shot run when an org opts in, plus a periodic scan for `classified_at IS NULL` rows. Newly-synced products before classification have `material=[], product_type=NULL` and behave as wildcards — pre-filter degrades gracefully (same path as `pre_filter_enabled=false`).

### Query-time extraction & filter

New module `apps/agent/src/agent/pre_filter.py`. **Per consultation proposal** (and once per regenerate-select run), a single LLM call (Pydantic structured output, low temperature) extracts `required_materials`, `required_product_types`, `required_heights` (fallback only — see below) and `color` from a single concatenated input string (`plan_markdown + proposal_description + user_modification`, whichever are non-empty).

`required_heights` are taken from `Geometry.fence_run_types[].height` when geometry is present (zero non-determinism), and from LLM extraction only when geometry has no typed fence runs. **All heights are stored and reasoned about in feet** — matching the geometry schema's units, so no unit conversion is needed anywhere in the pipeline.

Deterministic safety rules union baselines after extraction:

- `required_product_types ∪= {surface, post}` — every fence project needs these
- `required_product_types ⊇ {gate} ⇒ ∪= {gate_hardware}` — gates and gate hardware travel together
- `required_materials ⊇ {wood, vinyl, composite}` AND project mentions a gate ⇒ `∪= {chain_link}` — gate frames are galvanized steel
- Q&A mentions barbed wire AND `required_heights` non-empty ⇒ append `height − 1` foot per height (commercial 7+1 standard, e.g. 8ft fence with barbed top permits 7ft mesh)
- `required_materials = ∅` AND a fence-project context (defined as: `geometry.fence_run_types` non-empty OR any of plan/proposal/user_modification text non-empty) ⇒ treat as LLM error → **failed-open** (return full pool)

Filter routing per axis:

- **`material`** — keep when `c.material = ∅` OR `c.material ∩ required_materials ≠ ∅`.
- **`product_type`** — keep when `c.product_type ∈ {accessory, service, material}` (always-pass) OR `c.product_type ∈ required_product_types` OR `c.product_type IS NULL`.
- **`height`** — routed by `c.product_type`, skipped entirely when `required_heights = ∅`:
  - `surface` → `c.fence_height ∈ required_heights` exact match; NULL → keep
  - `gate` → `c.fence_height` OR `c.post_length` within ±1 ft of any required height; both NULL → keep
  - `post` → `c.post_length ≥ min(required_heights)`; NULL → keep. **Conscious choice: pre-filter ignores buried-depth.** The industry rule of thumb is roughly `post_length ≥ fence_height + 1/3 · post_length`, but actual buried depth varies by company, install method (dig / drive / surface mount), and region — `select_product/`'s catalog data shows even the standard pairing (6ft fence + 8' post, ~25% buried) violates a strict 1/3. Rather than encode an `install_method` signal at v1, pre-filter stays conservative: only short posts are dropped, and the downstream select LLM handles depth-vs-length nuances.
  - `gate_hardware` / `access_control` / `barbed_wire` / `accessory` / `service` / `material` → always keep (height-irrelevant)
- **`color`** — keep when `req.color = NULL` (customer didn't state) OR `c.product_type ∈ {gate_hardware, access_control, accessory, service, material}` (color is cosmetic for these — a black hinge fits a white fence) OR `c.color IS NULL` (untagged → wildcard) OR substring-match either direction (`req.color in c.color OR c.color in req.color`, lowercased — handles "Satin Black" ⊃ "Black" and similar).

The `search` node is **unchanged** — it still loads the org's full catalog into `state.candidates`. Filtering runs inside the executor nodes (`run_consultation_proposals` per-proposal inside the `asyncio.gather` fan-out, and `run_regenerate_select`). `run_regenerate_calculate` and `patch_bom` skip pre-filter entirely.

### Eval

v1 ships without a dedicated pre-filter eval suite. Pre-filter regressions surface through the existing E2E BOM eval's `missed_products`. A `pre_filter_recall` auxiliary metric is added to the evaluator's report for attribution. `eval/catalog.py` snapshot/restore is extended to cover the 5 new columns so eval runs against frozen classifications, not live LLM output.

## Considered alternatives

- **Restore embedding-based RAG / hybrid retrieval.** Rejected: same failure mode (top-K truncation losing cold-tail SKUs) that retired it. Set-intersection on closed-domain classes makes recall reasoning per-class instead of per-K.
- **Use `select_product/`'s 9-tag scheme directly.** Rejected: it conflates material (wood, vinyl) with functional role (`access_control`) on a single axis; LLM tag drift on 9 mixed-axis values is a known failure (`select_product/EXPERIMENT.md` Phase 7.5). Splitting into `material` (4 closed values, set-typed) and `product_type` (9 values, single-valued) keeps each axis tight.
- **Keep a separate `tags` set field alongside `material` and `product_type`.** Rejected: the values it would carry (`gate_hardware`, `access_control`, `barbed_wire`) are functional roles that fit cleanly inside `product_type` as accessory subclasses. Cross-axis-restricted SKUs (e.g. wood-only decorative hinge) are <2% of fence catalogs and fall back acceptably on the dominant role.
- **3-call union for query-time extraction (per `select_product`'s recall-stability mechanism).** Rejected for cost: per-proposal × 3 = up to 9 LLM calls per multi-proposal run. Replaced by single call + deterministic safety rules + failed-open; the safety rules absorb the recall-stability that union was buying.
- **Hook classification onto the catalog-sync path (`upsert_product_library`).** Rejected: would block the sync HTTP path on slow LLM calls. Async backfill (mirroring embeddings' established pattern) accepts a stale-window for new products in exchange for non-blocking sync.
- **SQL-side filter (PostgreSQL `&&` overlap, etc.) instead of Python in-memory filter.** Deferred to v2: 20k Candidates fetched into Python and filtered in-memory adds hundreds of ms but keeps the v1 `search` node and state-shape unchanged. Optimize if measured latency becomes a problem.
- **Dedicated pre-filter recall eval suite (mirroring `select_product`'s).** Deferred: maintaining double fixtures + ground-truth tagged catalogs is non-trivial. The auxiliary metric inside E2E BOM eval suffices for attribution at v1.
- **Post buried-depth aware filter** (1/3 industry rule, fixed +24" assumption, or `install_method` LLM extraction with dig/drive/surface mapping). Rejected at v1: every concrete formula either over-filters real catalogs (strict 1/3 drops the standard 6ft fence + 8' post pairing) or requires an `install_method` signal that the customer rarely states explicitly. The `post_length ≥ min(required_heights)` lower bound is the conservative compromise — short posts are dropped, depth-vs-length nuance is left to the select LLM. Revisit if a specific org reports systematic post-length miss patterns.
- **Inches as the storage unit for height/length** (matching select_product's industry-norm convention). Rejected: `Geometry.fence_run_types[*].height` is in feet, and forcing inches inside the filter pipeline would put a unit-conversion landmine on every read site. Storing in feet (Numeric(4,1)) keeps the whole pipeline unit-uniform.

## Out of scope

- **Non-fence orgs.** `material`'s value domain is fence-industry-closed; multi-vertical orgs would require extending the value domain in `CONTEXT.md` first.
- **`agricultural` and `composite` as `material` values.** Welded wire, T-post, Trex etc. are tagged with `material={chain_link}` or `{}` and routed through `product_type` (`accessory` / `surface`); recall on these is recovered via wildcard semantics rather than a dedicated `material` class. Revisit if specific orgs report systematic miss patterns.
- **Eval of tagging quality itself.** v1 freezes a manually spot-checked classification snapshot; whether the LLM's tagging is accurate is observed through production, not gated by eval.
- **Multi-valued `product_type`.** Single-valued today; cross-axis-restricted SKUs (<2%) handled by dominant-role assignment.

## Consequences

Additive change. Default `pre_filter_enabled=false` keeps non-opt-in orgs on today's full-catalog path. For opt-in orgs:

- **One-time classification cost** at opt-in: roughly $0.50 / 1k products and ~3 min for 1k, scaling linearly (~$3 and ~6 min for a 17k-product catalog, based on `select_product` numbers).
- **Per-query latency** rises by ~1 LLM call per proposal (~500 ms). Net latency typically improves: smaller candidate set → smaller select prompt → faster select call.
- **Stale window for newly-synced products** until classified — those products fall through as wildcards (precision degrades, never recall).
- **Schema migration is non-destructive.** Existing rows default to `material='{}'`, `product_type=NULL` — wildcard semantics match "pre-filter disabled".
- **Re-tagging on prompt v1 → v2** is gated by `classified_at < cutoff`. No version column needed; prompt upgrades are deliberate events.
- **Failed-open is load-bearing.** Any LLM extraction failure, sanity-check rejection, or DB issue degrades to the full-catalog path. Pre-filter cannot block BOM generation.
- **`patch_bom` is a known pre-filter blind spot** — it sees the unfiltered catalog. Acceptable: `patch_bom` operates by editing an existing BOM rather than selecting from scratch, so the candidate pool is mostly used as a list of replacement candidates.
