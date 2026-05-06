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

`documenso/` is also present but is a **vendored external project** (the open-source Documenso e-signing app — Remix + npm + Turbo). It is not part of ArcSite, has its own `CLAUDE.md`, and is gitignored at the workspace root. Treat it as reference/source-available code; do not change it as part of ArcSite work.

> The workspace root itself is a thin git repo that `.gitignore`s every sub-repo (including `documenso/`) — it only tracks this file and workspace metadata. `git status` / `git log` here will not reflect changes inside the sub-repos; `cd` into the sub-repo to use git.

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

Single-context: `CONTEXT.md` + `docs/adr/` at the workspace root. See `docs/agents/domain.md`.
