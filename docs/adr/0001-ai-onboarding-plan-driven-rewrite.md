# AI Onboarding v2: plan-driven flow

## Context

The original onboarding flow (`services/ai_onboarding/`) hardcoded fence-domain knowledge throughout the user experience — fixed question schemas per material (chainlink/vinyl/wood), bundle name conventions baked into prompts (`CL 4'H x 6'W S/G Res/Gal`), and an `install_method` (dig/drive) field in `bundles.jinja2` that fixed post lengths inside bundle data. The flow only worked end-to-end for three materials, prompts were long and example-heavy, and project-level concerns (concrete usage, post depth) leaked into catalog-level data (bundle composition).

## Decision

Rebuild around an LLM-authored **plan** that the user reviews before bundle generation. New pipeline: `parse → product gen → product review → LLM-generated questions → user answers → LLM-generated plan → user reviews / regenerates → execute (= rewritten gen_bundles) → bundle review → done`. The hardcoded `AnswerQuestions` page is deleted.

- **Plan shape**: `{strategy: markdown, bundle_names: string[]}`. No component recipe — `gen_bundles` produces components from `plan.bundle_names` + `plan.strategy` + `(Q, A)` pairs + `products` + `groups`.
- **Plan is read-only.** User revises only by writing a free-form instruction and triggering `regenerate_plan`, which re-runs the LLM with `(previous_plan, instruction)`. No per-field editing, no plan history.
- **Questions are LLM-authored**, schema `{id: slug, prompt, options?, multi?, optional?}` — three FE controls (radio / checkbox / textarea), 3–6 questions, required by default, no skip. The baseline-questions mechanism is kept in code but is **empty for all materials** — including no `install_method`, `section_length`, `gate_widths`. The LLM decides everything after seeing the parsed product library.
- **Bundles are install-agnostic.** `bundles.jinja2` no longer reads `use_concrete` or `underground_height`. Concrete / post depth become optional components inferred from the product list (presence of a concrete SKU implies an optional `Concrete` component on relevant bundles). Whether a project actually pours concrete is decided downstream when the bundle is consumed.
- **`gen_bundle_names` is removed.** `plan.bundle_names` is verbatim ground truth for execution; the LLM may not rename or invent bundles in the execute step.
- **`gen_product_groups` runs before plan generation** (not before bundle generation), so the plan's strategy can reference group names.

## Out of scope (intentionally preserved)

- **Per-material session structure** — a project still spawns one `OnboardingSession` per material sheet; questions / plan / execute are scoped per session.
- **`gen_products` and `categories/{wood,vinyl,chainlink}.py`** — hardcode here is acknowledged but not addressed in this iteration. `products.jinja2` is amended only to drop references to `material_answers` fields that no longer exist.
- **`Products.tsx` / `Bundles.tsx` review pages** and their feedback / pending / revert / regenerate APIs.
- **Backend stack** — all LLM calls remain in `cloudservice` with Gemini SDK + Jinja2 + Celery; migration to arc_agent's LangGraph + structured-output stack is deferred.

## Considered alternatives

- **Project-level plan that spans materials.** Rejected: per-material rules differ enough that cross-material bundles aren't useful for v2, and the change would force a rewrite of the per-session review flow.
- **Per-field plan editing** (user mutates `strategy` text or `bundle_names` list directly). Rejected: mixed user-edit + LLM contracts drift fast, and the user-edited strategy can become inconsistent with the bundle list. Chat-style regenerate keeps the LLM as the single source of truth.
- **Baseline questions for install method or section length.** Rejected: install method is project-level (one bundle should be reusable across dig and drive jobs); section length is usually inferable from product names; LLM judgment beats blanket asking.
- **Move `ai_onboarding` to arc_agent's LangGraph stack.** Deferred: PoW minimizes change to project/session/state-machine. Worth revisiting once the new flow is validated.

## Consequences

This is a **breaking change** with no data migration: `material_answers`, `bundle_names_failed_reason`, `FenceConfig` and its style literals, `gen_bundle_feasibility.precheck_*`, FE style/gate-width constants (`WOOD_FENCE_STYLES`, `single_gate_widths`, etc.), `StyleSelectionFormItem`, `RenameStyleModal`, `AnswerQuestions`, and `bundle_components_macro.jinja2` all go away. In-flight onboarding sessions from before the rewrite must be discarded.

First-execution bundle quality now depends entirely on prompt quality and the LLM's reading of the product library; unsatisfactory plans are routed through the user's regenerate loop instead of being silently masked by precheck or fence-specific defaults.

Future iterations can: (a) re-introduce baseline questions if specific gaps emerge in production, (b) add plan history / multi-turn conversation if users ask for it, (c) migrate LLM calls to arc_agent once stable, (d) revisit the hardcode that remains in `gen_products` / `categories/*.py`.
