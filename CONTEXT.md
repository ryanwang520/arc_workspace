# Domain Context

Glossary for the fence-industry product catalog as it shows up across the workspace. Terms here are canonical — when an issue, ADR, prompt, or model field names a domain concept, use the form below and avoid the listed aliases. Only fence-industry concepts and the cross-repo categorization scheme live here; framework/runtime concepts (LangGraph state, intent routing, BOM versioning) belong in each sub-repo's own docs.

## Language

### Catalog basics

**Product**:
A single SKU in an org's catalog. Belongs to exactly one **Org**.
_Avoid_: SKU, item, listing.

**Catalog**:
The full product list owned by one **Org**. A few hundred to ~20k. Authored in **cloudservice** (source of truth); **arc_agent** holds a read-only synced view used for BOM generation. The view is eventually consistent — see `docs/adr/0003-catalog-sync-snapshot-reconcile.md`.

**Org**:
A fence company using ArcSite. Each org has its own independent **Catalog**.

### Fence material systems

**Material**:
The physical fence construction system a **Product** is compatible with. Closed value domain: `wood`, `chain_link`, `vinyl`, `ornamental`. These are the four material systems a customer scopes a project around.
_Avoid_: "label", "tag", "category" — historical aliases that conflated material with functional role.

A **Product** carries a **set** of materials (`material: set[str]`):
- `{chain_link}` — used only in chain-link projects (e.g. BCL black mesh, galv. line post).
- `{wood, chain_link, vinyl, ornamental}` — universal hardware applicable across all systems (e.g. MagnaLatch).
- `{}` (empty) — project-level commodity (concrete, gravel, labor) with no material affinity.

`chain_link_galvanized` and `chain_link_color` are **not** distinct materials — both fall under `chain_link`. Coating is captured by `spec.color`.

### Product type

**Product type**:
The single role a **Product** plays in a fence project. Closed value domain (9 values), single-valued:

| Group | Values | Used by |
|---|---|---|
| Form factor (drives height filtering) | `surface`, `gate`, `post` | height filter |
| Functional role (Q&A-triggered) | `gate_hardware`, `access_control`, `barbed_wire` | functional filter |
| Catch-all | `accessory` | always passes both filters |
| Project-level overhead | `service`, `material` | always passes both filters |

`gate_hardware` covers hinges, latches, closers, drop rods. `access_control` covers gate operators, keypads, sensors. `barbed_wire` covers barbed/razor wire and top arms. `accessory` is the deliberate fall-through for products without a strong Q&A trigger (tension wire, post cap, privacy slats, ties). `service` is labor/install/demo/permit; `material` is concrete/gravel/sand.

### Pre-filter

**Pre-filter**:
An opt-in catalog reduction step that runs before the BOM-generation LLM sees candidates. Trims a large catalog (orgs above ~3k products) to a smaller candidate set, keyed on the customer's Q&A. Two axes: **Material** set intersection and **Product type** matching. Off by default per org; opt-in only.
_Avoid_: "粗筛" (loanword from prototype repo), "RAG" (the retired embedding-based retrieval was a different mechanism).

**Applicability semantics**:
Convention shared by both pre-filter axes: an **empty value on the product side counts as a wildcard** that matches any requirement. A product with `material: {}` is allowed in projects of any material; a product with `product_type: 'accessory'` is not blocked by functional filtering. The wildcard is on the product side, not on the requirement side.

## Relationships

- An **Org** owns a **Catalog** of **Products**.
- A **Product** has zero, one, or many **Material** values.
- A **Product** has exactly one **Product type**.
- **Pre-filter** keeps a **Product** when:
  - `material` is empty OR `material ∩ required_materials ≠ ∅`, AND
  - `product_type` is in the always-pass set (`accessory`, `service`, `material`) OR `product_type ∈ required_product_types`.

## Example dialogue

> **Dev:** "Customer says 'I want a 6ft black chain-link fence with one walk gate and an auto-opener.' What does pre-filter require?"
>
> **Domain expert:** "Required `material = {chain_link}`. Required `product_type = {surface, gate, post, gate_hardware, access_control}`. `barbed_wire` is not required, so the catalog's barbed-wire SKUs drop out. `accessory` always passes — tension wires and post caps come through on `material ∩ {chain_link}`."
>
> **Dev:** "And a wood-only decorative cedar hinge — does that survive?"
>
> **Domain expert:** "It would be tagged `material: {wood}, product_type: gate_hardware`. The customer's `required_materials = {chain_link}` doesn't intersect `{wood}`, so it drops out — correctly, since it physically can't fit a chain-link gate."

## Flagged ambiguities

- **"label" / "tag" / "category"** were used interchangeably in prototype work (`select_product/` uses 9 mixed-axis "labels"; `cloudservice/services/ai_onboarding/categories/` uses fence-style category folders). For the canonical pre-filter, only **Material** and **Product type** exist — both prior terms are dropped.
- **`material` (set) vs `product_type` (enum)** are both categorization but distinct axes. `material` is multi-valued because a product can be applicable to multiple systems; `product_type` is single-valued because a product has one physical/functional role. Don't merge them.
- **`access_control` is a `product_type`, not a material.** `select_product/`'s 9-tag scheme listed it alongside `wood`/`vinyl`, which conflated functional role with material system. Same correction for `barbed_wire`.
- **`universal` is not a value in either axis.** The "wildcard" semantic is carried by an empty `material` set (or the always-pass `product_type`s), not by a literal `universal` token.
