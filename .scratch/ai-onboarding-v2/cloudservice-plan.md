# cloudservice — AI Onboarding v2 backend plan

Read [PRD.md](./PRD.md) and [api-contract.md](./api-contract.md) first. Tasks are listed in dependency order.

Working directory: `/Users/ryanwang/code/arcsite/cloudservice`. Stack: Django + SQLAlchemy/Alembic, Celery, Gemini SDK, Jinja2. Lint/test: `just lint`, `just test`, `just migrate`.

---

## be-01 — Schema, models, migration

**Depends on**: `api-contract.md` committed.

**Files touched**:
- `arcservice/services/ai_onboarding/types.py`
  - Add `Question` and `Plan` Pydantic models per api-contract.
- `arcservice/models/products/onboarding.py`
  - Add columns to `OnboardingSession`: `questions: JSON | None`, `answers: JSON | None`, `plan: JSON | None`, `last_plan_instruction: Text | None`.
  - Drop columns: `material_answers`, `bundle_names_failed_reason`.
  - Add status enum members to `OnboardingStatus`: `QUESTIONS_GENERATING`, `QUESTIONS_GENERATED`, `QUESTIONS_GENERATE_FAILED`, `PLAN_GENERATING`, `PLAN_GENERATED`, `PLAN_GENERATE_FAILED`.
- `dbmigrate/versions/<timestamp>_ai_onboarding_v2.py` (alembic migration)
  - `op.add_column` for the 4 new columns.
  - `op.drop_column` for the 2 dropped columns.
  - Status is a string column (not enum-typed in DB), so no DDL needed for the new statuses.

**Acceptance**: `just migrate` runs cleanly forward and backward; `just lint` passes; reading/writing the new columns works in a manual SQLAlchemy shell session.

---

## be-02 — Remove FenceConfig and dead code

**Depends on**: be-01.

**Files touched**:
- DELETE `arcservice/services/ai_onboarding/config.py` (`FenceConfig` and style literals).
- DELETE `arcservice/services/ai_onboarding/gen_bundle_feasibility.py` (precheck).
- DELETE `arcservice/services/ai_onboarding/gen_bundle_names.py`.
- DELETE `arcservice/services/ai_onboarding/templates/bundle_components_macro.jinja2`.
- DELETE `arcservice/services/ai_onboarding/templates/bundle_names.jinja2` (if it exists).
- `arcservice/services/ai_onboarding/gen_products.py`
  - Drop `FenceConfig` parameter; signature accepts `material: str` and the products list (plus any retained kwargs that aren't fence-config-specific).
  - Update `get_session_config` callers in `feedback_ops.py` etc. — they shouldn't need a config anymore; refactor or pass `material` directly.
- `arcservice/services/ai_onboarding/templates/products.jinja2`
  - Remove all `{{ config.user_styles }}`, `{{ config.use_concrete }}`, `{{ config.section_length }}` etc. references.
  - Keep the `{{ material }}` reference and the `categories/<material>.py` data injection (categories are preserved per ADR).
- `arcservice/services/ai_onboarding/feedback_ops.py`
  - Update `apply_product_feedback_with_revision` and friends to call `gen_products` with the new signature.
- `arcservice/services/ai_onboarding/utils.py`
  - Remove or simplify `get_session_config` — it currently returns a `FenceConfig`.
- `arcservice/apps/home/modules/ai_onboarding/views.py`
  - In `start_session`: simplify `StartSessionPayload` to `{session_id: int}`; drop precheck call; drop `material_answers` write; transition `pending` → `products_generating`; `tasks.process_products.apply_async(...)`.
  - Remove all `FenceConfig` imports.
  - Remove `apply_product_feedback`'s use of `get_session_config`; pass material directly.
- `arcservice/services/ai_onboarding/tasks.py`
  - `ProductRecognitionPayload` no longer carries `material_answers`; `process_products` uses minimal context (material + products).
  - Remove any `gen_bundle_names` calls inside `generate_bundles_task` (this task is renamed/replaced in be-05; for now, just stop calling gen_bundle_names).
- Tests
  - Update or remove existing tests under `tests/` that mock `FenceConfig` or assert on `material_answers`. Some tests will be deleted entirely (precheck tests).
- `grep -rn "FenceConfig\|precheck_bundle_feasibility\|gen_bundle_names\|material_answers\|bundle_names_failed_reason" arcservice/` should return zero hits afterward.

**Acceptance**: `just lint` passes; `just test` passes (after test updates); existing code paths that don't depend on the deleted pieces still work.

---

## be-03 — `gen_questions` LLM call

**Depends on**: be-01.

**Files touched**:
- NEW `arcservice/services/ai_onboarding/gen_questions.py`
  - `def gen_questions(material: str, products: list[ProductDict]) -> list[Question]`.
  - Build the **b-view** of products: drop `sku`, `price`, `cost`, `description`; keep `name`, `category`, `unit`, `visual`, all dynamic attribute fields.
  - Render `templates/questions.jinja2`. Call Gemini (matching how `gen_products.py` calls it). Parse response with Pydantic structured output. Validate `id` is a slug (lowercase alphanumeric + underscore). Cap result at 6 entries; if LLM returns more, slice; if fewer than 3, fail.
- NEW `arcservice/services/ai_onboarding/templates/questions.jinja2`
  - System: "You're helping fence catalog setup. The sheet's material is `{{ material }}`. Below is the user's product library after categorization. Propose 3–6 questions that, combined with the products, will let you draft a sensible bundle plan. Skip anything obviously inferable from the products."
  - Show product b-view list as JSON.
  - Provide 1–2 generic example questions as soft hints (e.g. "What naming convention should I follow?", "Are there product groupings I should respect?"). NOT fence-specific.
  - Output schema: echo `Question` shape; English only.
- `arcservice/services/ai_onboarding/tasks.py`
  - NEW `tasks.generate_questions` Celery task.
    - Reads session, calls `gen_questions(session.material, session.products)`.
    - On success: `session.questions = [q.model_dump() for q in result]`; status → `questions_generated`.
    - On exception: status → `questions_generate_failed`.
- `arcservice/apps/home/modules/ai_onboarding/views.py`
  - NEW endpoint `POST /generate_questions/` per api-contract:
    - Validates session is in `products_generated` or `products_awaiting_confirmation`.
    - Sets status to `questions_generating`; kicks off `tasks.generate_questions`.
  - Update `get_session` response to include `questions` whenever `session.questions is not None`.
- `arcservice/apps/home/modules/ai_onboarding/urls.py`
  - Register the new route.

**Acceptance**: hitting `POST /generate_questions/` for a `products_generated` session returns 200; polling `GET /<session_id>/` yields `status: questions_generated` and a `questions` list matching the schema; an induced exception inside `gen_questions` lands the session in `questions_generate_failed`.

---

## be-04 — `gen_plan` LLM call (first + regenerate)

**Depends on**: be-01, be-02.

**Files touched**:
- NEW `arcservice/services/ai_onboarding/gen_plan.py`
  - ```python
    def gen_plan(
        material: str,
        products: list[ProductDict],
        groups: list[ProductGroup],
        questions: list[Question],
        answers: Answers,
        previous_plan: Plan | None = None,
        instruction: str | None = None,
    ) -> Plan: ...
    ```
  - Render `templates/plan.jinja2` with conditional `{% if previous_plan %}` block. Pydantic-validate the response. Reject responses where `bundle_names` are duplicated or empty.
- NEW `arcservice/services/ai_onboarding/templates/plan.jinja2`
  - Sections: material; products (b-view JSON); groups (JSON); Q/A pair list rendered explicitly; (if regenerate) previous plan + user instruction.
  - Output schema: echo `Plan` shape. Strategy is markdown prose 1–3 short paragraphs. `bundle_names` typically 5–30 unique entries.
- `arcservice/services/ai_onboarding/tasks.py`
  - NEW `tasks.generate_plan` Celery task:
    - If `session.product_groups` is empty, run `gen_product_groups` first (move/copy logic from existing tasks.py — currently lives in the old bundle generation chain).
    - Call `gen_plan(...)` with `previous_plan=None, instruction=None`.
    - On success: `session.plan = result.model_dump()`; status → `plan_generated`.
    - On exception: status → `plan_generate_failed`.
  - NEW `tasks.regenerate_plan` Celery task:
    - Same as above but pass `previous_plan = Plan.model_validate(session.plan)`, `instruction = session.last_plan_instruction`.
- `arcservice/apps/home/modules/ai_onboarding/views.py`
  - NEW endpoint `POST /submit_answers/`:
    - Validates session in `questions_generated`. Validates `answers` keys match `session.questions` ids.
    - Persists `session.answers`. Status → `plan_generating`. Kick off `tasks.generate_plan`.
  - NEW endpoint `POST /regenerate_plan/`:
    - Validates session in `plan_generated`. Validates `instruction` non-empty.
    - Persists `session.last_plan_instruction`. Status → `plan_generating`. Kick off `tasks.regenerate_plan`.
  - Update `get_session` response to include `answers`, `plan`, `last_plan_instruction` where api-contract says.
- `arcservice/apps/home/modules/ai_onboarding/urls.py`
  - Register two new routes.

**Acceptance**: full path products → questions → answers → plan works in a manual API call sequence. Regenerate produces a different but coherent plan that still references at least some bundle names from v1 (LLM keeping things it got right).

---

## be-05 — Bundle execute (rewrite `gen_bundles`)

**Depends on**: be-01, be-02, be-04.

**Files touched**:
- REWRITE `arcservice/services/ai_onboarding/templates/bundles.jinja2`
  - DROP all `{% if config.* %}` blocks.
  - DROP CL/Res/Gal/S/G fence-specific example blocks.
  - DROP the `{% import "bundle_components_macro.jinja2" as bundle_components %}` and the `bundle_components.render_bundle_components(...)` call.
  - ADD a "Plan" section near the top: render `{{ plan.strategy }}` and `{{ plan.bundle_names | tojson }}`. Tell the LLM "These bundle names are user-confirmed and must be used verbatim — do not rename, add, or remove."
  - ADD a "User context" section rendering the (question, answer) pair list.
  - KEEP the component matching rules (verbatim name match against products / groups / sub-bundles).
  - KEEP the quantity format rules (simple number / mathjs `INPUT` expression / `"X,Y,round up|down|no rounding"`).
  - KEEP the JSON output schema and the error-sample blocks.
- `arcservice/services/ai_onboarding/gen_bundles.py`
  - Refactor signature: `def gen_bundles(plan: Plan, questions: list[Question], answers: Answers, products: list[ProductDict], groups: list[ProductGroup], material: str) -> list[BundleDict]`.
  - Drop `FenceConfig` parameter and any `config.use_concrete` / `config.underground_height` branches.
  - Bundle names in the output **must** be a subset of `plan.bundle_names`; reject responses that introduce new names.
- `arcservice/services/ai_onboarding/tasks.py`
  - Rename existing `generate_bundles_task` → `execute_plan` (or add new and remove old).
    - Reads session; ensures `product_groups` is present (it should be from be-04 already, but defensively re-run if missing).
    - Calls `gen_bundles(...)` with the session's plan, questions, answers, products, groups, material.
    - Status: `plan_generated` → `bundles_generating` → `bundles_generated`.
- `arcservice/apps/home/modules/ai_onboarding/views.py`
  - NEW endpoint `POST /execute_plan/`.
  - DELETE the existing `generate_bundles` view function and the `revise_bundles` etc. behavior remains via existing endpoints.
- `arcservice/apps/home/modules/ai_onboarding/urls.py`
  - Register `/execute_plan/`. Remove `/generate_bundles/`.

**Acceptance**: `POST /execute_plan/` after a `plan_generated` session produces bundles whose `name`s match `plan.bundle_names` set-equally; quantity rules parse via `parse_quantity_rule`; review screen still renders bundles correctly.

---

## be-06 — Failure transitions

**Depends on**: be-01.

**Files touched**:
- `arcservice/apps/home/modules/ai_onboarding/views.py`
  - Extend `back_to_previous_status` to handle:
    - `questions_generate_failed` → `products_generated`
    - `plan_generate_failed` → `plan_generated` if `session.plan is not None` else `questions_generated`

**Acceptance**: hitting `back_to_previous_status` for each failed status transitions correctly.

---

## be-07 — Tests + manual e2e

**Depends on**: all above.

**Files touched**:
- NEW `tests/services/ai_onboarding/test_v2_happy_path.py`
  - Smoke test: mock Gemini, call `gen_questions` → assert returns valid `list[Question]`.
  - Smoke test: mock Gemini, call `gen_plan` (first) → assert returns valid `Plan`.
  - Smoke test: mock Gemini, call `gen_plan` (regenerate with previous + instruction) → assert returns valid `Plan` distinct from previous.
  - Smoke test: mock Gemini, call `gen_bundles` → assert names ⊆ `plan.bundle_names`.
- (Optional) Integration test that exercises the endpoint chain with mocked Celery + Gemini.
- Manual demo against the CDN xlsx template per PRD §"Demo target".

**Acceptance**: `just test` passes; manual demo flow completes without exceptions.
