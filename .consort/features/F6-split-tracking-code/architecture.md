# F6 — Architecture (S1-expand-split-columns)

Layering inherited from F1 (boundary=app/routes react, service=app/services, repository=app/repositories, models=app/models/, infrastructure=app/database.py). All S1 ACs are **Infra**: an additive Alembic expand migration plus a data-integrity probe. No route, no API contract, no UI in S1.

## Architectural Concerns Mapping

| Concern | Owner layer | Module | Notes |
|---|---|---|---|
| Schema change + backfill | Infrastructure (migration) | alembic/versions/<expand> | Nullable batch_number/serial_number; in-DB split guarded by 3-non-empty-segment predicate |
| Domain mapping of new columns | Models | app/models/stock_record.py | `batch_number`, `serial_number: str | None` |
| Integrity probe query | Repository | app/repositories/tracking_code_integrity.py | Only layer touching the session |
| Probe report (warning) | Service | app/services/tracking_code_integrity.py | `{level:'warning', nonconforming_count}`; `python -m` entrypoint |
| Transactions | Migration transaction | alembic | add + backfill atomic (PI3) |
| Validation / auth | N/A | — | No new input path; auth out of bounds |
| Config | Env | DATABASE_URL | Branch-provisioned databricks_postgres |
| Logging | Service | probe | Prints the warning with the count |

## Pattern proposals
- Expand-then-contract: S1 expand (add+backfill, tracking_code kept), S2 read switch, S3 contract (drop).
- Split rule lives once in SQL in the migration; the probe reuses the same conformance predicate (single source in the repository module) so count and backfill cannot disagree.

## Risks
- Predicate drift between migration and probe — mitigated by sharing the predicate definition and testing both against the same seed.
- Probe reads tracking_code, so it lapses at S3 (OD3, deferred).

## Decisions
- OD1: probe warns, does not block — recommended + resolved.
- OD2: >3 segments, empty segments, NULL codes are nonconforming — recommended + resolved per spec.

## Test strategy
Real pytest-bdd integration tests against the paired Lakebase branch (no mocks/in-memory): seed rows at the prior revision, `alembic upgrade head`, assert AC1/AC2/AC3; downgrade/upgrade round-trip for PI2; constraint probes for NFR2/NFR3; AC4 via the probe service against seeded data. FK-aware targeted DELETE cleanup.

## Sign-off
Proceed — architect-reviewer.

## S2-show-batch-serial (Batch/Serial display)

| AC | Layer | Owner | Concerns |
|---|---|---|---|
| AC1-home-table-batch-serial-columns | E2E | client/ stock table + EP1 GET /api/stock | client↔server contract; serialization at boundary |
| AC2-home-table-not-tracked | E2E | client/ stock table + EP1 | null→"not tracked" rendering in client components |
| AC3-detail-shows-batch-serial | E2E | client/ SKU detail page + EP2 GET /api/skus/{sku} | client↔server contract |
| AC4-detail-not-tracked | E2E | client/ SKU detail page + EP2 | null→"not tracked" rendering |

- Read-only story: extends `StockRecordOut`/`SkuLocationOut` (app/routes/) with nullable `batch_number`/`serial_number` read via existing `StockService` → `app/repositories/stock.py` → `app/models/stock_record.py`. No migration, no new persistence invariant, no new access pattern (same queries, extra columns).
- SPA never splits `tracking_code`; the server columns are the single source (NFR7). Tests: Playwright e2e against the live API on the paired branch (seeded rows incl. 'X-1' nonconforming), plus API pytest-bdd for the EP1/EP2 contract.
- Risk: S3 contract drops `tracking_code`; EP1/EP2 keep it until then (OD3 still deferred).
- Sign-off (S2): proceed — architect-reviewer.

## S3-contract-drop-tracking-code (contract step)

All three ACs are `Infra`: the contract Alembic migration (down_revision = S1 expand) drops `stock_records.tracking_code`; `app/models/stock_record.py` loses the column; `tracking_code` leaves EP1/EP2 response items; the S1 integrity probe module is retired (OD3 resolved). Downgrade rebuilds `tracking_code` as `location-batch-serial` where both batch and serial are non-NULL, else NULL. New invariants PI5 (reversible, rows survive), PI6 (transactional drop), PI7 (constraints retained); NFR8 (brief_ref R1) covers AC2/AC3.

- **Risks:** the downgrade cannot recover original nonconforming codes (they become NULL) nor codes whose leading segment differed from `location`; accepted as the documented reverse. Any stray reader of `tracking_code` (client, route, probe) breaks after the drop — the layering/contract tests must run with the column gone.
- **Decisions:** OD3 — retire the probe at contract (recommended); re-pointing it to NULL batch/serial counts would duplicate S2's "not tracked" display.
- **Test strategy:** pytest-bdd on the paired Lakebase branch: upgrade + information_schema + parent-aware branch schema diff (AC1), pre/post snapshot (AC2), downgrade/upgrade round-trip (AC3). No mocks.
- **Sign-off:** proceed — architect-reviewer.
