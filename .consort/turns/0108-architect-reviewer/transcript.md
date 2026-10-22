# architect-reviewer (architect-reviewer) – opus

## Prompt

```
Annotate story S2-show-batch-serial's acceptance criteria + nfrs.md coverage. Author architectural_notes for ALL of story S2-show-batch-serial's ACs: AC1-home-table-batch-serial-columns, AC2-home-table-not-tracked, AC3-detail-shows-batch-serial, AC4-detail-not-tracked. Every one of this story's ACs MUST end up with a non-empty "architectural_notes" field in its acs/<AC>.json (the layer it lives in + how it realizes the design) — the design gate verifies that, and the spec-author's "layer" field does NOT count. architectural_notes are per-AC, so author the scoped ACs above even when the feature-level architecture.json already exists from an earlier story. For every E2E AC, ALSO pass consort-annotate-ac --ui-seams with the ia.md data-testid seam token(s) the AC's UI drives — the column / cell / region / empty-state / form field it asserts (e.g. "stock-cell-tracking,stock-not-tracked"). A deterministic gate checks each seam is named in <PROJECT_ROOT>/.consort/design/ia.md; if the AC needs a surface the IA does not yet name, declare the seam you need anyway and the gate routes the ux-designer to add it to ia.md. In architecture.json, make an EXPLICIT service_backed call (required): set service_backed:true if the feature persists data (a DB table/migration) or carries business logic, and then you MUST declare boundary, service, and repository layers (plus a "models" PACKAGE app/models/, one module per domain object, NOT a flat app/models.py, when it persists entities); set false ONLY for a trivial static/read-through endpoint. An Infra-layer AC or a migration/schema/storage NFR while service_backed is false hard-blocks the gate. When service_backed:true you MUST also declare architecture.json persistence_invariants[]: the DB-level guarantees the schema enforces (each with id, type one of unique|foreign_key|cascade|not_null|check|transactional|migration_reversible, table, and a one-line brief), covering unique/composite keys, foreign keys + cascade rules, NOT NULL / CHECK constraints, any transactional-atomicity boundary, and migration reversibility. The test-strategist must cover each with a real-branch test; a service_backed feature with no persistence_invariants hard-blocks the gate. When THIS story introduces a NEW read/query shape over the schema — a filter, lookup, sort, or join an AC drives (e.g. "list stock filtered by location", "look up a SKU's rows by sku") that is an INDEX candidate — declare it in architecture.json access_patterns[] (id, a one-line brief, and the table + column(s) it reads) so the DBA is pulled back in to reconsider the physical design for it; this is how a later READ story evolves the schema, since a story that adds no persistence_invariant would otherwise skip the DBA. Declare an access_pattern ONLY for a genuinely new query shape (not for every read, and not for one already covered by a prior story's pattern or a primary-key lookup). REUSE the established project architecture conventions (set by F1-stock-visibility): boundary=app/routes (react), service=app/services, repository=app/repositories, models=app/models, infrastructure=app/database.py. Declare the SAME role -> module paths in architecture.json, do NOT remap or rename an established layer; a divergent layout hard-blocks the spec gate and mismatches the inherited code. Author this story's architectural_notes FRESH from the ACs + IA on disk this turn. On a reflect-revise re-run the self-heal clears this story's prior architectural_notes BY DESIGN, so an absent/empty prior is the expected revise state — re-derive against the corrected ACs + IA. Write every artifact under the ABSOLUTE artifact root <PROJECT_ROOT>/.consort (this feature: <PROJECT_ROOT>/.consort/features/F6-split-tracking-code/; this story: <PROJECT_ROOT>/.consort/features/F6-split-tracking-code/stories/S2-show-batch-serial/); use that absolute path and never resolve or guess the project root yourself.CROSS-STORY CONTEXT — this story is NOT reviewed in isolation. The feature's OTHER stories' ACs, the architecture's open_decisions, and its mandated (not-null) persistence fields are below; use them for the cross-story checks (an AC contradicting a gated sibling AC; a mandated field no sibling submit AC supplies). This IS that context — do not run `consort-cross-story-context` yourself.

{
  "current_story": "S2-show-batch-serial",
  "sibling_stories": [
    {
      "story": "S1-expand-split-columns",
      "acs": [
        {
          "ac_id": "AC1-conforming-code-backfilled",
          "status": "draft",
          "layer": "Infra",
          "given": "existing stock records whose tracking_code parses as location-batch-serial by the '-' delimiter, at varying widths (e.g. 'A12-B7-S001', 'L1-BATCH22-9')",
          "when": "the expand migration is applied",
          "then": "each record's batch_number holds the second segment and serial_number the third segment of its tracking_code (e.g. 'B7' / 'S001'), and tracking_code is still present and unchanged",
          "architectural_notes": "Infra: a data-store contract realized by an ADDITIVE Alembic expand migration (alembic/versions/) that adds nullable stock_records.batch_number/serial_number (mapped on app/models/stock_record.py) and backfills them in-database via string_to_array(tracking_code,'-') only where exactly three non-empty segments exist (segment 2 -> batch, 3 -> serial), leaving tracking_code untouched. Cross-cutting: migration reversibility + transactional backfill (PI3/PI4) owned by the migration; verified by a pytest-bdd test that seeds the paired branch at the prior revision, upgrades, and reads rows back — no mock DB."
        },
        {
          "ac_id": "AC2-nonconforming-code-left-null",
          "status": "draft",
          "layer": "Infra",
          "given": "existing stock records whose tracking_code does not parse as location-batch-serial (e.g. 'X-1', a bare 'c') or that have no tracking_code",
          "when": "the expand migration is applied",
          "then": "those records keep their batch_number and serial_number NULL (no guessed or partial value) and remain present with their tracking_code unchanged",
          "architectural_notes": "Infra: the same expand migration's non-parse branch — its backfill UPDATE is guarded by the three-non-empty-segment predicate, so fewer/more segments, an empty segment, or a NULL code leave batch_number/serial_number NULL (no partial split) and the row plus its tracking_code are untouched (PI1 nullable columns, no not_null). Owner: the migration in alembic/versions/; verified on the paired branch by seeding 'X-1', 'c', 'A-B-C-D', 'A--C' and NULL codes before upgrade."
        },
        {
          "ac_id": "AC3-every-row-survives-unchanged",
          "status": "draft",
          "layer": "Infra",
          "given": "the full set of sprint-1 stock records before the change",
          "when": "the expand migration is applied",
          "then": "the same number of stock records exists afterwards and each record's sku, location, and quantity are identical to before; location is never rewritten from the tracking_code's leading segment",
          "architectural_notes": "Infra: the R1 row-preservation guarantee for an ADDITIVE migration on the pre-existing stock_records table — the backfill only SETs batch_number/serial_number, never location/sku/quantity, and never INSERTs/DELETEs, so count and canonical fields are identical; UNIQUE(sku,location) and CHECK(quantity>=0) remain in force (NFR2/NFR3). Owner: the expand migration; verified on the paired branch by snapshotting (sku,location,quantity) before upgrade and comparing after, plus a downgrade/upgrade round-trip that keeps every row (PI2)."
        },
        {
          "ac_id": "AC4-integrity-probe-reports-nonconforming",
          "status": "draft",
          "layer": "Infra",
          "given": "the expand migration has been applied to data containing a known number of nonconforming tracking codes",
          "when": "the integrity probe is run against the paired branch",
          "then": "it reports a warning with the exact count of stock records whose tracking_code did not parse as location-batch-serial, so the PO can review it before accepting the change",
          "architectural_notes": "Infra: a data-integrity probe over stock_records — app/services/tracking_code_integrity.py (service) builds a warning report {level:'warning', nonconforming_count:n} from a COUNT query in app/repositories/tracking_code_integrity.py (the only layer touching the session); runnable as python -m app.services.tracking_code_integrity against the branch DATABASE_URL. It warns and never blocks (OD1 resolved); nonconforming follows the feature-spec definition incl. NULL codes (OD2 resolved); verified by a pytest-bdd test seeding a known nonconforming count on the paired branch."
        }
      ]
    }
  ],
  "open_decisions": [
    {
      "id": "OD1-probe-blocks-or-warns",
      "question": "Does the integrity probe block acceptance or only warn?",
      "decision_status": "resolved",
      "resolved_by_story": "S1-expand-split-columns",
      "resolution": "Warn with the exact count; PO accepts at review (per feature request)."
    },
    {
      "id": "OD2-nonconforming-definition",
      "question": "Are >3-segment codes and NULL codes nonconforming (NULL batch/serial, counted by the probe)?",
      "decision_status": "resolved",
      "resolved_by_story": "S1-expand-split-columns",
      "resolution": "Yes per feature-spec Data semantics: only exactly three non-empty '-' segments conform; 'A-B-C-D', empty segments and NULL codes are nonconforming and counted."
    },
    {
      "id": "OD3-probe-after-contract",
      "question": "Once S3 drops tracking_code, is the probe retired or re-pointed (it reads tracking_code)?",
      "decision_status": "deferred"
    }
  ],
  "required_persistence_fields": []
} Be terse: produce ONLY the required artifact file(s) on disk, then stop with at most a one-line confirmation. Do NOT print a plan, a summary of what you did, rationale, tables, or restate the artifacts to stdout, that output is wasted latency. The files on disk are the deliverable, not your prose.
```

## Tools used

- Bash {"command":"cd <PROJECT_ROOT>/.consort/features/F6-split-tracking-code && cat stories/S2-show-batch-serial/acs/*.json && cat architecture.json && grep -n -i \"batch\\|serial\\|tracking\\|not-tracked\" ../../design/ia.md | head -60","description":"Read S2 ACs, architecture, and IA seams"}
- Bash {"command":"cd <PROJECT_ROOT> && grep -rn \"@router\\.\\(get\\|post\\)\\|APIRouter(\" app/routes/ | head; ls app/services app/repositories","description":"List existing API routes and service modules"}
- Bash {"command":"sed -n 1,60p app/routes/stock.py; sed -n 1,40p app/routes/skus.py; ls .consort/features/F6-split-tracking-code/","description":"Inspect stock and SKU route signatures"}
- Bash {"command":"cd <PROJECT_ROOT>/.consort/features/F6-split-tracking-code && python3 - <<'EOF'\nimport json\np='architecture.json'; a=json.load(open(p))\na['api_contracts']=[c for c in a['api_contracts'] if c['id'] not in ('EP1-list-stock-batch-serial','EP2-sku-detail-batch-serial')]\na['api_contracts']+= [\n {\"id\":\"EP1-list-stock-batch-serial\",\"method\":\"GET\",\"path\":\"/api/stock\",\"query\":[\"location\"],\"shape\":\"list\",\"item\":\"StockRecordOut {sku: string, location: string, quantity: int, tracking_code: string|null, batch_number: string|null, serial_number: string|null}; null batch/serial = not tracked\",\"story\":\"S2-show-batch-serial\"},\n {\"id\":\"EP2-sku-detail-batch-serial\",\"method\":\"GET\",\"path\":\"/api/skus/{sku}\",\"shape\":\"object\",\"item\":\"SkuDetailOut {sku: string, locations: [{location: string, quantity: int, tracking_code: string|null, batch_number: string|null, serial_number: string|null}]}; null batch/serial = not tracked\",\"story\":\"S2-show-batch-serial\"}]\na['nfrs']=[n for n in a['nfrs'] if n['id']!='NFR7-not-tracked-never-blank']\na['nfrs'].append({\"id\":\"NFR7-not-tracked-never-blank\",\"tier\":\"product\",\"applies_to\":\"S2-show-batch-serial\",\"hil_status\":\"accepted\",\n \"requirement\":\"The API exposes batch_number/serial_number as nullable fields (never a fabricated value); the SPA renders a NULL field as 'not tracked' per field, never blank, without parsing tracking_code client-side.\",\n \"fitness_function\":\"On the paired branch, a stock record with NULL batch_number and populated serial_number is returned by GET /api/stock and GET /api/skus/{sku} with batch_number null and serial_number set (no client-side split of tracking_code).\"})\njson.dump(a,open(p,'w'),indent=2)\nEOF\ncat >> architecture.md <<'EOF'\n\n## S2-show-batch-serial (Batch/Serial display)\n\n| AC | Layer | Owner | Concerns |\n|---|---|---|---|\n| AC1-home-table-batch-serial-columns | E2E | client/ stock table + EP1 GET /api/stock | client↔server contract; serialization at boundary |\n| AC2-home-table-not-tracked | E2E | client/ stock table + EP1 | null→\"not tracked\" rendering in client components |\n| AC3-detail-shows-batch-serial | E2E | client/ SKU detail page + EP2 GET /api/skus/{sku} | client↔server contract |\n| AC4-detail-not-tracked | E2E | client/ SKU detail page + EP2 | null→\"not tracked\" rendering |\n\n- Read-only story: extends `StockRecordOut`/`SkuLocationOut` (app/routes/) with nullable `batch_number`/`serial_number` read via existing `StockService` → `app/repositories/stock.py` → `app/models/stock_record.py`. No migration, no new persistence invariant, no new access pattern (same queries, extra columns).\n- SPA never splits `tracking_code`; the server columns are the single source (NFR7). Tests: Playwright e2e against the live API on the paired branch (seeded rows incl. 'X-1' nonconforming), plus API pytest-bdd for the EP1/EP2 contract.\n- Risk: S3 contract drops `tracking_code`; EP1/EP2 keep it until then (OD3 still deferred).\n- Sign-off (S2): proceed — architect-reviewer.\nEOF","description":"Add S2 API contracts, NFR, and architecture notes"}
- Bash {"command":"cd <PROJECT_ROOT> && F=F6-split-tracking-code; S=S2-show-batch-serial\n./scripts/lk consort-annotate-ac --feature $F --story $S --ac AC1-home-table-batch-serial-columns --layer E2E --api-contract EP1-list-stock-batch-serial --ui-seams \"stock-table,stock-col-batch,stock-col-serial,stock-batch-<sku>-<loc>,stock-serial-<sku>-<loc>\" --notes \"E2E: client↔server contract — the React home stock table (client/ pages+components) renders batch_number/serial_number from GET /api/stock in separate Batch and Serial columns, dropping the combined tracking column. Boundary app/routes/stock.py serializes the nullable columns read via StockService -> app/repositories/stock.py -> app/models/stock_record.py; no client-side split of tracking_code. Verified by Playwright against the live API on the paired branch.\" &&\n./scripts/lk consort-annotate-ac --feature $F --story $S --ac AC2-home-table-not-tracked --layer E2E --api-contract EP1-list-stock-batch-serial --ui-seams \"stock-table,stock-batch-<sku>-<loc>,stock-serial-<sku>-<loc>,stock-batch-not-tracked-<sku>-<loc>,stock-serial-not-tracked-<sku>-<loc>\" --notes \"E2E: the server returns NULL batch_number/serial_number for a nonconforming code (S1 backfill); the React stock table maps each null field independently to a muted 'not tracked' cell, never blank, while the populated sibling still renders. Null-to-label presentation is owned by the client component; the boundary only passes nulls through (NFR7). Verified by Playwright on the paired branch with a seeded 'X-1' row.\" &&\n./scripts/lk consort-annotate-ac --feature $F --story $S --ac AC3-detail-shows-batch-serial --layer E2E --api-contract EP2-sku-detail-batch-serial --ui-seams \"sku-detail,sku-locations,sku-location-<loc>,sku-batch-<loc>,sku-serial-<loc>\" --notes \"E2E: client↔server contract — the React SKU detail page renders each position's batch_number/serial_number from GET /api/skus/{sku} as separately labelled Batch and Serial fields replacing sku-tracking-<loc>. app/routes/skus.py SkuLocationOut gains the nullable fields sourced via StockService.sku_locations -> app/repositories/stock.py. Verified by Playwright against the live API on the paired branch.\" &&\n./scripts/lk consort-annotate-ac --feature $F --story $S --ac AC4-detail-not-tracked --layer E2E --api-contract EP2-sku-detail-batch-serial --ui-seams \"sku-location-<loc>,sku-batch-not-tracked-<loc>,sku-serial-not-tracked-<loc>\" --notes \"E2E: GET /api/skus/{sku} passes NULL batch_number/serial_number through unchanged; the React detail page renders each null field as 'not tracked' for that field alone, never blank. Presentation owned by the client component, data by the existing service/repository path (NFR7). Verified by Playwright on the paired branch with a seeded nonconforming row.\" &&\n./scripts/lk consort-response-formatter --role architect-reviewer --feature $F --story $S","description":"Annotate S2 ACs and run self-check"}

## Final reasoning

I've tagged all four S2 ACs as E2E, each with architectural notes, UI seams from `ia.md` and an API contract. The self-check passes.

- **`architecture.json`:**
  - Two new API contracts, both carrying nullable `batch_number` and `serial_number`:
    - `EP1-list-stock-batch-serial`: GET `/api/stock`, returns a bare list.
    - `EP2-sku-detail-batch-serial`: GET `/api/skus/{sku}`, returns an object.
  - One new product NFR, `NFR7-not-tracked-never-blank`: a missing batch or serial shows as "not tracked", never blank, and the client never splits `tracking_code` itself.
- **No schema changes:** S2 adds no migration, no persistence invariant and no access pattern, since it reads the same queries with two extra columns.
- **`architecture.md`:** I appended an S2 section with the mapping table, the test approach (Playwright against the live API on the paired branch, plus API tests for both endpoints), and a "proceed" sign-off.
