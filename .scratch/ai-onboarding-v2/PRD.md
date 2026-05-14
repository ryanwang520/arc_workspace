# AI Onboarding v2 — Implementation PRD

This is the execution PRD for the rewrite described in [ADR-0001](../../docs/adr/0001-ai-onboarding-plan-driven-rewrite.md). Read the ADR first; this document only covers how to execute.

## Documents in this folder

- [`PRD.md`](./PRD.md) — this file. Goal, parallel strategy, demo target.
- [`api-contract.md`](./api-contract.md) — single source of truth for endpoints, payloads, status enum, JSON shapes. **Backend and frontend both read from this.**
- [`cloudservice-plan.md`](./cloudservice-plan.md) — backend tasks with order and dependencies (`be-01` … `be-07`).
- [`arcsite_web-plan.md`](./arcsite_web-plan.md) — frontend tasks with order and dependencies (`fe-01` … `fe-09`).

## What we're building

A plan-driven onboarding flow. The user uploads a product spreadsheet; the LLM analyses the parsed products and proposes a small set of dynamic questions; the user answers; the LLM generates a `Plan` (strategy markdown + bundle name list) the user can revise via free-form regenerate instructions; the user executes the plan; bundles are produced and reviewed via the existing review flow. The hardcoded `AnswerQuestions` page and the `FenceConfig` / `gen_bundle_names` paths are removed. See ADR-0001 for the full design rationale.

## Parallel execution strategy

Phases are gated by cross-repo dependencies; tasks within a phase are independent and run in parallel.

| Phase | Backend (cloudservice) | Frontend (arcsite_web) |
|-------|------------------------|------------------------|
| 0 | `api-contract.md` drafted and committed | (waits) |
| 1 | be-01 schema + migration | fe-01 types + api client; fe-02 react-markdown |
| 2 | be-02 remove FenceConfig + dead code | fe-03 StartSessionPage; fe-04 DynamicQuestions; fe-05 PlanReview (with mocked plans) |
| 3 | be-03 gen_questions; be-04 gen_plan | fe-06 orchestrator; fe-07 Products + status components |
| 4 | be-05 execute_plan / bundles.jinja2 rewrite | fe-08 delete dead components |
| 5 | be-06 failure transitions | (cleanup if any) |
| 6 | be-07 tests + demo | fe-09 manual e2e |

FE Phase 2–3 work uses **mocked API responses** matching `api-contract.md`. They don't wait for BE; they integrate during Phase 6.

If a single contributor takes both swim lanes, they should still work top-to-bottom within each lane (`be-01 → be-02 → be-03 …`) because each task assumes the previous one is in place.

## Out of scope (this iteration)

See ADR-0001 §"Out of scope". Briefly: per-material session structure, `gen_products` and `categories/{material}.py`, bundle review/feedback flow, backend stack — all preserved.

## Demo target

After Phase 6, demo the full happy path against the existing template:
`https://cdn-public.arcsiteapp.com/Upload+Products+to+ArcSite.xlsx`

Demo story (single material, e.g. wood):

1. Navigate to `/fence/onboarding`.
2. Upload the xlsx → confirm modal → land on project page.
3. Open the `wood` session → click "Start Analysis".
4. Products are auto-generated; review (no edits).
5. Click "Generate Plan" → 3–6 dynamic questions appear.
6. Answer them → submit.
7. Review the generated plan (strategy paragraph + bundle name list).
8. (Optional) Regenerate with an instruction like "please use shorter bundle names" or "drop all 4ft variants".
9. Click "Execute plan" → bundles generate.
10. Review bundles → confirm → see the success page.

Capture a recording for team review.
