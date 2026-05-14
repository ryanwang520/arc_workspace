# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this directory is

`arc_workspace/` is **not a monorepo** — it is a workspace folder that colocates several independent git repositories. There is no top-level `package.json`, `Makefile`, `pyproject.toml`, or shared tooling here. Run commands from inside the relevant sub-repo.

The four ArcSite sub-repos:

| Sub-repo            | Role                                                                                      | Stack                                                   |
| ------------------- | ----------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| `arc_agent/`        | LangGraph BOM generation agent + FastAPI gateway + React playground                       | Python 3.13 (uv), Bun, PostgreSQL + pgvector            |
| `arcsite_web/`      | Frontend monorepo: `web`, `web2`, `h5`, `enterprise`, email `server`, shared `packages/*` | pnpm workspace + Turbo, React 18/19, Vite/Next.js       |
| `cloudservice/`     | ArcSite main cloud backend + React `admin-frontend/`                                      | Django + SQLAlchemy/Alembic (uv), MySQL; admin uses Bun |
| `arc_dep_cluster/`  | Ansible + Docker Swarm deployment/infra for the whole application stack (test / prod)     | Ansible, Docker Swarm, AWS ECR, Graylog, Datadog        |

Other top-level directories — each its own independent git repo, gitignored at the workspace root, **not** part of the four ArcSite sub-repos:

| Directory            | Role                                                                                                                                                                                              |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `skills/`            | Workspace's custom Claude Code skills (`engineering/` + `html-artifacts/` buckets). Shared content under `html-artifacts/_shared/` is propagated into each skill by `scripts/sync-shared.sh`. See its own `CLAUDE.md` for layout rules. |
| `artifacts-gallery/` | Internal hub for publishing the HTML produced by the `html-*` skills above. Bun + Hono backend + React 19 SPA, own `deploy.yml`. Has its own `CONTEXT.md` (Artifact / Revision / Author) — independent from the workspace-root one. |
| `select_product/`    | **Retired** product pre-filter prototype (FastAPI + Postgres, ~21k products across 4 onboarded fence companies). Informed ADR 0002 but is **not active code** — its architecture was borrowed and its terminology (`粗筛`, 9 mixed-axis "labels", `chain_link_galvanized` vs `chain_link_color`) was deliberately dropped. Kept for historical reference only.                |
| `documenso/`         | **Vendored external project** (open-source Documenso e-signing app — Remix + npm + Turbo). Not part of ArcSite; has its own `CLAUDE.md`. Treat as reference / source-available code; do not modify as part of ArcSite work.                            |

> The workspace root itself is a thin git repo that `.gitignore`s every sub-repo and every directory above — it only tracks workspace-level files (`CLAUDE.md`, `CONTEXT.md`, `docs/adr/`). `git status` / `git log` here will not reflect changes inside any of them; `cd` into the relevant directory to use git.

Each sub-repo has its own `CLAUDE.md` (symlinked to `AGENTS.md`) — **read the nested one for anything repo-specific**:

- `arc_agent/CLAUDE.md` — agent graph flow, eval commands, change checklists for follow-up / retrieval / memory
- `arcsite_web/CLAUDE.md` — pnpm-only rule, per-app API client conventions, styling and React-hook rules
- `cloudservice/CLAUDE.md` — `just` command reference, service/apps/core layering, import conventions
- `arc_dep_cluster/CLAUDE.md` — Swarm service → node-label mapping, `make deploy-test` / `make deploy-prod`, `roles/deploy/templates/docker-compose.yml` as the source of truth

## Working across sub-repos

- **cd into the sub-repo first.** Package managers, lockfiles, venvs, and test runners are scoped per sub-repo and do not compose. `uv`, `bun`, `pnpm`, `just`, `ansible`, and `make` each exist in some but not all of them.
- **Different package managers are load-bearing:** `cloudservice` uses `uv` + `just` + `bun` (admin only); `arc_agent` uses `uv` + `bun` + `make`; `arcsite_web` uses `pnpm` exclusively (never npm/yarn); `arc_dep_cluster` uses `ansible` + `make`.
- **Cross-repo contracts are informal.** The agent's mobile endpoint (`POST /api/ai/fence/propose_products` in `arc_agent/apps/api`) is consumed by ArcSite mobile clients; `cloudservice` is the upstream auth/data service that `arc_agent` and `arcsite_web` both integrate with; `arc_dep_cluster` deploys the built images of the other three but is driven by git-branch/tag conventions, not a shared build system. There is no shared type system between them — check both ends when changing a contract.
- **Don't leak changes across repos.** A change in one sub-repo should not touch files in another; each has independent history, CI, and deploy (`deploy.yml` per sub-repo). The application sub-repos each ship their own `deploy.yml`; `arc_dep_cluster` owns stack-level orchestration (Swarm services, node labels, nginx/redis/graylog playbooks).

## Agent skills

### Issue tracker

Issues live as markdown files under `.scratch/<feature>/` at the workspace root. See `docs/agents/issue-tracker.md`.

### Triage labels

Default canonical vocabulary (`needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`), written as a `Status:` line at the top of each issue file. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: `CONTEXT.md` + `docs/adr/` at the workspace root. See `docs/agents/domain.md`. Current ADRs:

- `0001-ai-onboarding-plan-driven-rewrite.md` — replaces hardcoded fence question schemas with an LLM-authored plan reviewed by the user before bundle generation (`cloudservice/services/ai_onboarding/`).
- `0002-product-pre-filter.md` — opt-in catalog pre-filter for large orgs (5k–20k SKUs) in `arc_agent`, using closed-domain set classification (`material` × `product_type`) plus height/color routing. Canonical for the pre-filter design.
- `0003-catalog-sync-snapshot-reconcile.md` — Celery-beat snapshot reconcile from cloudservice → arc_agent with soft-delete (`deleted_at`); replaces the manual per-row upsert sync.
