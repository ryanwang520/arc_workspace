# AI Onboarding v2 — API contract

Single source of truth for backend ↔ frontend coordination. When this document and the code disagree, the document is wrong — update it.

All endpoints require login (`@api_required_login`) and live under `user/service/ai_onboarding/`.

---

## Status enum

`OnboardingSession.status` is a lower-cased string. Full set after v2:

```
pending
products_generating
products_regenerating
products_regenerated
products_awaiting_confirmation
products_generate_failed
products_generated
questions_generating          ← new
questions_generated           ← new
questions_generate_failed     ← new
plan_generating               ← new (also used during regenerate)
plan_generated                ← new
plan_generate_failed          ← new
bundles_generating
bundles_generate_failed
bundles_generated
bundles_regenerating
bundles_regenerate_failed
bundles_regenerated
bundles_awaiting_confirmation
product_library_generating
product_library_generate_failed
product_library_generated
```

`pending` still exists; it just transitions to `products_generating` via `start_session` instead of via `start_session+answers`.

---

## Pydantic / TS types

### Question

```python
class Question(BaseModel):
    id: str                          # slug, e.g. "preferred_heights"
    prompt: str                      # natural-language question shown to user
    options: list[str] | None = None # null → free-text; non-null → selectable
    multi: bool = False              # only meaningful when options is non-null
    optional: bool = False           # default required
    allow_other: bool = False        # only meaningful when options is non-null;
                                     # see "On allow_other" below
    recommended: str | list[str] | None = None  # see "On recommended" below
```

```ts
type Question = {
  id: string;
  prompt: string;
  options?: string[];
  multi?: boolean;
  optional?: boolean;
  allow_other?: boolean;
  recommended?: string | string[] | null;
};
```

#### On `allow_other`

When `options` is non-null and `allow_other = true`, the FE renders an extra `"Other"` choice alongside the listed options. Selecting `"Other"` reveals a free-text input below; that text is required (non-empty, non-whitespace) before submit.

The LLM-side `questions.jinja2` prompt is responsible for telling the model **not** to write a literal `"Other (please specify)"` string into `options` — it must use the `allow_other` flag instead. FE/BE never round-trip the literal string `"Other"` as an answer.

#### On `recommended`

The LLM proposes a default answer for each question — the answer it would pick if the user accepted its analysis as-is. The FE surfaces this as a hint **without auto-applying it**, so the user has to make an explicit choice on every question (this catches cases where the recommendation is wrong, and keeps answers genuinely user-driven instead of AI-driven):

- `multi: false` + `options` set → `recommended` is a string that **must** appear in `options`. FE labels that option with `(recommended)` but does **not** pre-select it. Sentinel strings like `"Other"` are forbidden.
- `multi: true` + `options` set → `recommended` is a list of strings, each one in `options`. FE labels each one with `(recommended)` but does **not** pre-check them.
- Free-text question (`options` null) → `recommended` is a draft string. FE shows it as a TextArea placeholder and exposes a "Use AI suggestion" link that fills the field on click. The value is **not** auto-pre-filled — the user must commit it.

`recommended` is optional. The LLM may omit it (or set `null`) when there's no defensible default. Recommendations the FE can't validate (e.g. an option-set value that's not in `options`) are silently ignored.

### Answers

```python
Answers = dict[str, str | list[str]]
# keyed by Question.id
# value is str when free-text or single-select (multi=false)
# value is list[str] when multi=true
```

```ts
type Answers = Record<string, string | string[]>;
```

When `allow_other = true` and the user selects "Other":
- single-select: the answer is the user's free text (the literal string `"Other"` is never submitted).
- multi-select: the user's free text is appended to the list of selected options as one entry.

### Plan

```python
class Plan(BaseModel):
    strategy: str                    # markdown, LLM-authored prose
    bundle_names: list[str]          # unique, non-empty, drives execute verbatim
```

```ts
type Plan = {
  strategy: string;
  bundle_names: string[];
};
```

---

## Endpoints

### Modified

#### `POST /start_session/`
- **Request**: `{session_id: int}` (was `{session_id, answers}`)
- **Behavior**: requires `pending` status; transitions to `products_generating`; triggers `tasks.process_products`.
- **Response**: `{session_id, status, material}`

### New

#### `POST /generate_questions/`
- **Request**: `{session_id: int}`
- **Behavior**: requires `products_generated` (or `products_awaiting_confirmation`); transitions to `questions_generating`; triggers `tasks.generate_questions`.
- **Response**: `{session_id, status}`

#### `POST /submit_answers/`
- **Request**: `{session_id: int, answers: Answers}`
- **Behavior**: requires `questions_generated`; persists `session.answers`; transitions to `plan_generating`; triggers `tasks.generate_plan` (which ensures `gen_product_groups` has run before calling `gen_plan`).
- **Response**: `{session_id, status}`

#### `POST /regenerate_plan/`
- **Request**: `{session_id: int, instruction: str}` (`instruction` must be non-empty)
- **Behavior**: requires `plan_generated`; persists `session.last_plan_instruction`; transitions to `plan_generating`; triggers `tasks.regenerate_plan` with `previous_plan = session.plan` and the new instruction.
- **Response**: `{session_id, status}`

#### `POST /execute_plan/`
- **Request**: `{session_id: int}`
- **Behavior**: requires `plan_generated`; transitions to `bundles_generating`; triggers `tasks.execute_plan` (the rewritten gen_bundles).
- **Response**: `{session_id, status}`

### Deleted

- `POST /generate_bundles/` — replaced by `/execute_plan/`. URL routing removed.

### Unchanged

All other existing endpoints stay verbatim:

`/projects/`, `/projects/<id>/`, `/start_project/`, `/<session_id>/` (get_session), `/save_product_feedback/`, `/save_bundle_feedback/`, `/apply_product_feedback/`, `/update_product_feedback/`, `/update_bundle_feedback/`, `/regenerate_products/`, `/regenerate_bundles/`, `/<session_id>/revert_products/`, `/<session_id>/revert_bundles/`, `/<session_id>/confirm_products/`, `/<session_id>/confirm_bundles/`, `/<session_id>/apply_all_pending_product_changes/`, `/<session_id>/apply_all_pending_bundle_changes/`, `/<session_id>/go_back/`, `/back_to_previous_status/`, `/generate_product_library/`, `/retry/`, `/project/<id>/reupload_product_sheet/`, `/project/<id>/confirm/`.

`/back_to_previous_status/` gains two new transitions:
- `questions_generate_failed` → `products_generated`
- `plan_generate_failed` → `plan_generated` if `session.plan` is non-null, else `questions_generated`

---

## `get_session` response (`GET /<session_id>/`)

Always includes: `session_id`, `project_id`, `status`, `material`.

| Status | Additional fields |
|--------|-------------------|
| `pending` | (none) |
| `products_generating` / `products_regenerating` | (none) |
| `products_generated` / `products_regenerated` / `products_awaiting_confirmation` | (existing v1 fields: `products`, `product_feedback`, `pending_product_feedback`, `product_changes`, `product_revertible`, `products_ever_regenerated`) |
| `questions_generating` | (none) |
| `questions_generated` | `questions: Question[]` |
| `plan_generating` | `questions: Question[]`, `answers: Answers`, `plan: Plan \| null` (null on first generation, prev plan on regenerate), `last_plan_instruction: string \| null` |
| `plan_generated` | `questions: Question[]`, `answers: Answers`, `plan: Plan`, `last_plan_instruction: string \| null` |
| `bundles_*` | (existing v1 bundle review fields) |
| `product_library_generated` | `result: { ... }` (existing v1) |
| any `*_failed` | `error?: { message: string }` (best-effort) |

`material_answers` and `bundle_names_failed_reason` are removed from the response.

---

## OnboardingSession DB columns

### Added (all nullable)
- `questions JSON` — `list[Question]` once persisted
- `answers JSON` — `Answers` (dict)
- `plan JSON` — `Plan` (dict)
- `last_plan_instruction TEXT`

### Removed
- `material_answers JSON`
- `bundle_names_failed_reason TEXT`

### Untouched
All other columns (`products`, `bundles`, `product_groups`, `product_feedback`, `bundle_feedback`, `pending_*`, `*_changes`, `*_revertible`, `result`, `task_id`, status, FK columns) — unchanged.
