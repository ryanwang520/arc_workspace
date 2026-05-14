# Catalog sync via snapshot reconcile

## Context

cloudservice is the source of truth for every Org's **Catalog** (see `CONTEXT.md`); arc_agent needs a queryable copy because BOM generation runs `fetch_all_candidates(org_id)` against `arcsite_shared.models.Product` rows in its own Postgres. Until now the sync was a single helper `sync_product_library_to_agent(org_id)` that POSTed `{uuid, name, unit}` per row to `POST /internal/orgs/{org_id}/product_library` — invoked manually from the django shell. Four problems blocked productionisation: (1) no automated trigger, (2) no scoping to orgs that actually use AI BOM (gated by per-org feature flag `CAN_USE_FENCE_BIM_OBJECT`), (3) deletes on cloudservice never propagated — products lingered in arc_agent forever, (4) the upsert endpoint had no "this is the full list" semantic so it physically could not express deletes.

## Decision

**Snapshot reconcile** driven by a Celery beat task in cloudservice. Each tick:

1. Reverse-query `FeatureLimit` for orgs with `CAN_USE_FENCE_BIM_OBJECT` enabled (per-org grants only — this flag is never granted at `user_level`).
2. For each enabled org, build a full payload of every non-deleted, non-history `Product` (`{uuid, name, unit}` — no other business fields for v1) and `PUT /internal/orgs/{org_id}/product_library/snapshot`.
3. arc_agent treats the request as authoritative: rows with `uuid` in the payload are upserted; rows for that `org_id` whose `uuid` is **not** in the payload get `deleted_at = NOW()`. A previously-deleted `uuid` reappearing in the payload clears `deleted_at` (restore by uuid).
4. Single-org failures are logged + emitted to Datadog and skipped — no in-cycle retry; the next scheduler tick self-heals.

Schema additions on arc_agent's `products`:
- `deleted_at TIMESTAMPTZ NULL` — soft-delete marker. All catalog read sites (`fetch_all_candidates`, name reconciliation, backfill scans, eval snapshot) filter `WHERE deleted_at IS NULL`. Soft delete also preserves any per-row metadata that downstream services compute (e.g. embeddings, future classification columns) so accidental delete + restore does not re-pay that cost.

Endpoint surface:
- **New**: `PUT /internal/orgs/{org_id}/product_library/snapshot` — full-snapshot semantics (handles upsert + soft-delete + restore in one transaction).
- **Removed**: `POST /internal/orgs/{org_id}/product_library` — the upsert-only endpoint had only one caller (cloudservice's sync helper), which moves to the new PUT in the same change. No external consumers, no transition window needed.

Off → on flag transitions trigger an immediate one-shot sync via a hook in `feature_limit_svc.add_or_update_limit_for_org` (the only writer for per-org `CAN_USE_FENCE_BIM_OBJECT`); the scheduler still owns the steady-state. On → off transitions are passive: the scheduler simply stops including the org. arc_agent retains the org's product rows (no purge) — the cloudservice-side `CAN_USE_FENCE_BIM_OBJECT` gate already prevents the BOM agent from being invoked for that org, so retained rows are inert. Re-enabling the flag is reversible without re-paying any per-row metadata cost.

## Considered alternatives

- **Event-driven push** — hook every Product create/update/delete site in cloudservice to call a partial-update endpoint. Rejected: `product_svc.py` and adjacent services have many write paths (direct CRUD, XLS import, bundle/option-set cascades, restore). Each missed hook produces a silent divergence that is invisible until a customer reports a stale BOM. Snapshot reconcile is correct-by-construction — the next tick reconciles whatever drift happened.
- **Hard delete on snapshot diff** — `DELETE FROM products WHERE org_id=X AND uuid NOT IN (...)`. Rejected: (a) destroys any per-row metadata downstream services compute (embeddings today; classification columns if pre-filter is reintroduced), (b) in-flight BOM runs that already loaded a candidate set become inconsistent if a row vanishes mid-flight, (c) makes "why did this BOM reference a SKU that no longer exists?" un-debuggable. Soft delete preserves audit trail and degrades safely.
- **Hook the off → on trigger inside the scheduler** (diff "previously enabled orgs" set, treat new arrivals as immediate sync). Rejected because `CAN_USE_FENCE_BIM_OBJECT` is granted only via the single `add_or_update_limit_for_org` write path — a direct hook there is simpler than maintaining a separate "previously enabled" snapshot in redis. Would have been preferred if the flag could also flip via `user_level` upgrades, but it cannot.
- **Expand sync payload to include `description`, `price`, `hide_in_app`, etc.** Deferred: keeps v1 small. `description` is the most tempting addition (would improve LLM context for selection) but most customer catalogs do not populate it consistently — revisit when that changes. `price` is already fetched live from cloudservice by mobile clients, so syncing it would create a stale-price hazard.
- **Retry on single-org failure** within the same tick. Rejected for v1: the next scheduled tick is the natural retry, and cascading retries on a wider arc_agent outage produce a thundering herd. Datadog alerting on failure rate is sufficient.

## Consequences

- **Staleness window**: new / updated / deleted products on cloudservice become visible to the agent on the next scheduler tick (target: hourly). Real-time visibility is not a goal of this design.
- **Snapshot payload size scales linearly** with catalog size. A 17k-product org sends ~2 MB JSON per tick × N enabled orgs. Will need pagination or delta-by-`update_time` if either dimension grows substantially; deferred until measured.
- **arc_agent's `products` table grows monotonically** for any org that has ever had churn — soft-deleted rows accumulate. A GC job (e.g. hard-delete `deleted_at < now() - interval '90 days'`) is deferred until table size becomes a concern.
- **Snapshot reconcile is the only sync path.** The PUT endpoint always interprets its payload as authoritative — sending a partial list will silently soft-delete every uuid not present. The endpoint contract must be enforced by callers; there is no server-side guard.
