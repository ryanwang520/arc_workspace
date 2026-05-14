# arcsite_web — AI Onboarding v2 frontend plan

Read [PRD.md](./PRD.md) and [api-contract.md](./api-contract.md) first. Tasks are listed in dependency order.

Working directory: `/Users/ryanwang/code/arcsite/arcsite_web/apps/web`. Stack: pnpm workspace, Vite, React 18, Ant Design, react-query.

Type-check: `pnpm tsc --noEmit` from `apps/web`. Lint: `pnpm lint`. Dev server: `pnpm dev`.

---

## fe-01 — Types + API client

**Depends on**: `api-contract.md` committed.

**Files touched**:
- `apps/web/src/features/ai-onboarding/types.ts`
  - Add `Question`, `Plan` types per api-contract.
  - Add new statuses to the `OnboardingStatus` union: `'questions_generating' | 'questions_generated' | 'questions_generate_failed' | 'plan_generating' | 'plan_generated' | 'plan_generate_failed'`.
  - Extend `OnboardingSessionStatus` with optional `questions?: Question[]`, `answers?: Record<string, string | string[]>`, `plan?: Plan | null`, `last_plan_instruction?: string | null`.
  - Drop `material_answers`, `bundle_names_failed_reason` from `OnboardingSessionStatus` (and any helper types).
- `apps/web/src/features/ai-onboarding/api.ts`
  - Add `generateQuestions(sessionId)`, `submitAnswers(sessionId, answers)`, `regeneratePlan(sessionId, instruction)`, `executePlan(sessionId)`.
  - Simplify `startSession` signature to `startSession(sessionId: string)` (no answers).
  - Remove `generateBundles`.

**Acceptance**: `pnpm tsc --noEmit` passes; `pnpm lint` passes.

---

## fe-02 — Add `react-markdown`

**Depends on**: nothing (independent).

**Files touched**:
- `apps/web/package.json` — add `react-markdown` (and optionally `remark-gfm` if we want tables/checklists in strategy).
- Run `pnpm install` from monorepo root.

**Acceptance**: `import ReactMarkdown from 'react-markdown'` resolves without TS errors.

---

## fe-03 — `StartSessionPage`

**Depends on**: fe-01.

**Files touched**:
- NEW `apps/web/src/features/ai-onboarding/pages/onboarding-session/StartSessionPage.tsx`
  - Props: `{ material: string; sessionId: string; projectId: string; onBack: () => void }`.
  - Layout: page title (capitalized material), short paragraph explaining what happens next ("We'll analyze your products, propose questions, then generate a plan you can review"), a primary "Start Analysis" button, a Back button.
  - Click "Start Analysis" → mutation calling `startSession(sessionId)` → on success invalidate `['onboarding', sessionId]` query so the orchestrator polls and switches view.

**Acceptance**: rendered standalone (a temporary route or Storybook), clicking the button calls the mutation and triggers the network request.

---

## fe-04 — `DynamicQuestions`

**Depends on**: fe-01.

**Files touched**:
- NEW `apps/web/src/features/ai-onboarding/components/DynamicQuestions.tsx`
  - Props: `{ material: string; sessionId: string; questions: Question[]; onSubmitted: () => void }`.
  - Wrap an antd `<Form/>`; for each `question`:
    - `options == null` → `<Input.TextArea rows={3}/>` with `rules={[{ required: !question.optional, message: 'Please answer this question' }]}`.
    - `options != null && !multi` → `<Radio.Group options={options}/>` with required-rule.
    - `options != null && multi` → `<Checkbox.Group options={options}/>` with required-rule (`min: 1`).
  - Submit handler: build `Answers` dict keyed by `question.id`, call `submitAnswers(sessionId, answers)` mutation; on success invalidate session query and call `onSubmitted()`.

**Acceptance**: rendered with a fixture `questions` array — all three control types display, validation triggers on empty submit, success calls `submitAnswers`.

---

## fe-05 — `PlanReview`

**Depends on**: fe-01, fe-02.

**Files touched**:
- NEW `apps/web/src/features/ai-onboarding/components/PlanReview.tsx`
  - Props: `{ sessionId: string; plan: Plan; lastInstruction: string | null }`.
  - Local state: `instruction: string` (controlled textarea).
  - Layout (single column, max-w-720):
    1. **Strategy** card: header "Plan strategy"; render `plan.strategy` via `<ReactMarkdown>`.
    2. **Bundle names** card: header "Bundles to generate (N)"; antd `<List>` showing one name per line.
    3. **Action** card: header "Refine or execute":
       - `<Input.TextArea/>` for `instruction`. Placeholder: `"e.g., 'Please remove all 4ft variants'  /  'Use SKU prefix CL- for chainlink bundles'  /  'Add a double walk gate option'"`.
       - "Regenerate plan" button (primary-ghost): disabled when `instruction.trim() === ''`; clicking → `regeneratePlan(sessionId, instruction)` mutation → invalidate session query.
       - "Execute plan" button (primary): clicking opens an antd `Modal.confirm` ("Execute plan? We'll generate N bundles based on this plan. You can still review and edit them afterward."); confirming → `executePlan(sessionId)` mutation → invalidate session query.
  - If `lastInstruction` is non-null, show a small read-only block above the textarea: "Last instruction: …" — purely informational.

**Acceptance**: rendered with a fixture plan; markdown renders correctly; regenerate disabled on empty input; execute fires immediately.

---

## fe-06 — Update `OnboardingSession` orchestrator

**Depends on**: fe-03, fe-04, fe-05.

**Files touched**:
- `apps/web/src/features/ai-onboarding/pages/onboarding-session/OnboardingSession.tsx`
  - Replace the `pending` branch's `<AnswerQuestions>` render with `<StartSessionPage material={...} sessionId={...} projectId={...} onBack={...}/>`.
  - Add new branches:
    - `'questions_generating'` → `<GeneratingStateResult status={status} ...>`
    - `'questions_generated'` → `<DynamicQuestions material={...} sessionId={...} questions={session.questions!} onSubmitted={...}/>`
    - `'plan_generating'` → `<GeneratingStateResult status={status} ...>`
    - `'plan_generated'` → `<PlanReview sessionId={...} plan={session.plan!} lastInstruction={session.last_plan_instruction ?? null}/>`
    - `'questions_generate_failed'`, `'plan_generate_failed'` → `<FailedStateResult status={status} ...>` (handled in fe-07).
  - Update the polling status array (`pollingStates`) to include `'questions_generating'`, `'plan_generating'`.

**Acceptance**: navigating into a session in each new state renders the right component; polling continues during the new generating states.

---

## fe-07 — Update `Products` + status components

**Depends on**: fe-01.

**Files touched**:
- `apps/web/src/features/ai-onboarding/components/Products.tsx`
  - `onNext` callback now triggers `generateQuestions(sessionId)` mutation (replacing the old `generateBundles`).
  - Button label: "Generate Bundles" → "Generate Plan".
- `apps/web/src/features/ai-onboarding/components/GeneratingStateResult.tsx`
  - Add status messages for `questions_generating` ("Drafting questions for you…") and `plan_generating` ("Drafting your plan…"). Reuse existing layout.
- `apps/web/src/features/ai-onboarding/components/FailedStateResult.tsx`
  - Add `questions_generate_failed` ("Question generation failed") and `plan_generate_failed` ("Plan generation failed") to the `failedStates` map.
  - Both back / retry buttons hit `back_to_previous_status` — same pattern as existing failures.

**Acceptance**: visual review on each generating / failed state; clicking "Generate Plan" on the products review page triggers the new endpoint.

---

## fe-08 — Delete dead components

**Depends on**: fe-06.

**Files touched**:
- DELETE `apps/web/src/features/ai-onboarding/components/AnswerQuestions.tsx`.
- DELETE `apps/web/src/features/ai-onboarding/components/StyleSelectionFormItem.tsx`.
- DELETE `apps/web/src/features/ai-onboarding/components/RenameStyleModal.tsx`.
- Clean up imports in `OnboardingSession.tsx`, any references in `types.ts` (`StyleError`, hardcoded `WoodFenceStyle` etc.) and `utils.ts`.
- `grep -rn "AnswerQuestions\|StyleSelectionFormItem\|RenameStyleModal\|WOOD_FENCE_STYLES\|single_gate_widths\|double_gate_widths\|StyleError\b" apps/web/src/features/ai-onboarding` should return zero hits.

**Acceptance**: `pnpm tsc --noEmit` and `pnpm lint` pass.

---

## fe-09 — Manual e2e

**Depends on**: all FE + BE done.

Demo flow per PRD §"Demo target":

1. `pnpm dev` from `apps/web`; open the running URL.
2. Navigate to `/fence/onboarding`.
3. Upload `Upload+Products+to+ArcSite.xlsx` (downloaded from the existing CDN link).
4. Confirm modal → land on the project page.
5. Pick the `wood` session → click "Start Analysis".
6. Wait for product generation; review (no edits, just verify a few rows).
7. Click "Generate Plan".
8. Answer 3–6 dynamic questions.
9. Review the plan — strategy markdown + bundle names list both visible.
10. (Optional) Try `regenerate` with an instruction such as "please use shorter bundle names".
11. Click "Execute plan".
12. Wait for bundle generation; review.
13. Click "Confirm" → see the success page.

Capture a screen recording for team review.

**Acceptance**: full happy path completes without console errors; recording uploaded somewhere the team can see.
