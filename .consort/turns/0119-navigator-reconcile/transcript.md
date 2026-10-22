# navigator-reconcile (navigator) – sonnet

## Prompt

```
FEATURE-COMPLETE RECONCILIATION for F6-split-tracking-code: every story is now designed + gated. Independently check the WHOLE set of stories against ITSELF — symmetrically, every story vs every other, INCLUDING the first-designed story (which was never checked against its later siblings). The cross-story context (all stories' ACs + the architecture's open_decisions + its mandated not-null fields) is injected below. Look ONLY for CROSS-STORY defects: an AC that contradicts another story's gated AC; a mandated (not-null) field no story's submit AC supplies; two stories' ACs overlapping on the same observable behavior; a persistence invariant anchored to the wrong story. Do NOT re-review a single story in isolation — each already passed its own reflect + gate. Write the verdict to <PROJECT_ROOT>/.consort/features/F6-split-tracking-code/reconcile-verdict.json: {"version":1,"passed":<bool>,"findings":[{"owner":"spec-author"|"test-strategist"|"architect-reviewer"|"ux-designer","story":"<the OWNING story to change>","detail":"<the cross-story defect>","ac_ids":["<exact AC id(s)>"],"severity":"blocking"|"advisory"}]}. Use passed:true with findings:[] when the stories are mutually consistent. Each finding names the OWNING story (the one reopened + changed) and its author. Produce ONLY the verdict file, then stop.FEATURE-COMPLETE CROSS-STORY RECONCILIATION — every story below is now designed + gated. Check the WHOLE set against ITSELF, symmetrically (every story vs every other, including the first-designed one): an AC that contradicts another story's gated AC; a mandated (not-null) field no story's submit AC supplies; two stories' ACs that overlap on the same observable behavior; an invariant anchored to the wrong story. For each defect, name the OWNING story (the one to change) + its author (owner). This IS the context — do not run `consort-cross-story-context` yourself.

{
  "stories": [
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
    },
    {
      "story": "S2-show-batch-serial",
      "acs": [
        {
          "ac_id": "AC1-home-table-batch-serial-columns",
          "status": "draft",
          "layer": "E2E",
          "given": "stock records at a location whose batch and serial are known (e.g. batch 'B7', serial 'S001')",
          "when": "the operator opens the home stock-by-location table",
          "then": "each row shows separate Batch and Serial columns holding that record's batch ('B7') and serial ('S001'), and the table no longer shows a single combined tracking-code column",
          "architectural_notes": "E2E: client↔server contract — the React home stock table (client/ pages+components) renders batch_number/serial_number from GET /api/stock in separate Batch and Serial columns, dropping the combined tracking column. Boundary app/routes/stock.py serializes the nullable columns read via StockService -> app/repositories/stock.py -> app/models/stock_record.py; no client-side split of tracking_code. Verified by Playwright against the live API on the paired branch."
        },
        {
          "ac_id": "AC2-home-table-not-tracked",
          "status": "draft",
          "layer": "E2E",
          "given": "a stock record whose batch and/or serial has no value (e.g. its original code was nonconforming, such as 'X-1')",
          "when": "the operator opens the home stock-by-location table",
          "then": "each empty field's cell reads 'not tracked' in its own column (Batch, Serial), never blank, while any populated field in the same row still shows its value",
          "architectural_notes": "E2E: the server returns NULL batch_number/serial_number for a nonconforming code (S1 backfill); the React stock table maps each null field independently to a muted 'not tracked' cell, never blank, while the populated sibling still renders. Null-to-label presentation is owned by the client component; the boundary only passes nulls through (NFR7). Verified by Playwright on the paired branch with a seeded 'X-1' row."
        },
        {
          "ac_id": "AC3-detail-shows-batch-serial",
          "status": "draft",
          "layer": "E2E",
          "given": "a SKU with stock records whose batch and serial are known",
          "when": "the operator opens that SKU's detail view",
          "then": "each stock position on the detail view shows its Batch and Serial as distinct, separately labelled fields with that record's values, in place of the combined tracking code",
          "architectural_notes": "E2E: client↔server contract — the React SKU detail page renders each position's batch_number/serial_number from GET /api/skus/{sku} as separately labelled Batch and Serial fields replacing sku-tracking-<loc>. app/routes/skus.py SkuLocationOut gains the nullable fields sourced via StockService.sku_locations -> app/repositories/stock.py. Verified by Playwright against the live API on the paired branch."
        },
        {
          "ac_id": "AC4-detail-not-tracked",
          "status": "draft",
          "layer": "E2E",
          "given": "a SKU with a stock record whose batch and/or serial has no value",
          "when": "the operator opens that SKU's detail view",
          "then": "each empty field reads 'not tracked' for that field (Batch, Serial) on the detail view, never blank",
          "architectural_notes": "E2E: GET /api/skus/{sku} passes NULL batch_number/serial_number through unchanged; the React detail page renders each null field as 'not tracked' for that field alone, never blank. Presentation owned by the client component, data by the existing service/repository path (NFR7). Verified by Playwright on the paired branch with a seeded nonconforming row."
        }
      ]
    },
    {
      "story": "S3-contract-drop-tracking-code",
      "acs": [
        {
          "ac_id": "AC1-tracking-code-dropped",
          "status": "draft",
          "layer": "Infra",
          "given": "the expand step has been applied (batch_number and serial_number exist and are backfilled) and nothing reads tracking_code",
          "when": "the contract migration is applied and the parent-aware schema diff is run against the paired branch",
          "then": "the stock table no longer has a tracking_code column, and the diff shows tracking_code removed alongside batch_number and serial_number added",
          "architectural_notes": "Infra: a data-store contract realized by the CONTRACT Alembic migration (alembic/versions/, down_revision = the S1 expand revision) that drops stock_records.tracking_code and removes the column from app/models/stock_record.py; precondition is zero readers — the S1 integrity probe is retired and tracking_code leaves the EP1/EP2 response items in the same change (OD3 resolved). Verified by a pytest-bdd test on the paired branch that inspects information_schema after upgrade and runs the parent-aware branch schema diff, asserting tracking_code removed and batch_number/serial_number added versus the parent."
        },
        {
          "ac_id": "AC2-rows-survive-contract",
          "status": "draft",
          "layer": "Infra",
          "given": "the full set of stock records after the expand step, including records with NULL batch/serial from nonconforming codes",
          "when": "the contract migration is applied",
          "then": "the same number of stock records exists and each record's sku, location, quantity, batch_number, and serial_number are identical to before the drop",
          "architectural_notes": "Infra: the R1 row-preservation guarantee for the contract migration — a pure DROP COLUMN in one migration transaction (PI6) that never INSERTs/DELETEs/UPDATEs, so row count and every (sku, location, quantity, batch_number, serial_number) incl. NULL batch/serial rows are unchanged, with UNIQUE(sku,location) and CHECK(quantity>=0) retained (PI7). Owner: the contract migration; verified on the paired branch by snapshotting those fields at the expand revision, upgrading, and comparing — no mock DB."
        },
        {
          "ac_id": "AC3-round-trip-reversible",
          "status": "draft",
          "layer": "Infra",
          "given": "the contract migration has been applied to records with known location, batch_number, and serial_number",
          "when": "the contract migration is reversed and then re-applied",
          "then": "the reverse step restores a tracking_code built as location-batch-serial (e.g. 'A12-B7-S001') with every row present, and re-applying drops tracking_code again so the branch ends on the up-migrated schema without tracking_code",
          "architectural_notes": "Infra: migration reversibility of the contract step (PI5) — downgrade re-adds nullable tracking_code and rebuilds it in-database as location||'-'||batch_number||'-'||serial_number only where both batch and serial are non-NULL (otherwise NULL; the original nonconforming code is not recoverable), keeping every row; upgrade head drops it again. Owner: the contract migration in alembic/versions/; verified by a pytest-bdd round-trip on the paired branch asserting the rebuilt code (e.g. 'A12-B7-S001'), row count, and the final head schema without tracking_code."
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
      "decision_status": "resolved",
      "resolved_by_story": "S3-contract-drop-tracking-code",
      "resolution": "Retired: the probe is a pre-acceptance check for the expand step (S1 AC4, gated before S3); the contract change deletes app/services|repositories/tracking_code_integrity.py and drops tracking_code from EP1/EP2 so nothing reads the column. S1 AC4 still holds at the expand revision."
    }
  ],
  "required_persistence_fields": []
} Be terse: produce ONLY the required artifact file(s) on disk, then stop with at most a one-line confirmation. Do NOT print a plan, a summary of what you did, rationale, tables, or restate the artifacts to stdout, that output is wasted latency. The files on disk are the deliverable, not your prose.
```

## Tools used

- Write {"file_path":"<PROJECT_ROOT>/.consort/features/F6-split-tracking-code/reconcile-verdict.json","content":"{\"version\":1,\"passed\":true,\"findings\":[]}\n"}

## Final reasoning

Wrote `reconcile-verdict.json` with `passed: true` and no findings. I found no cross-story defects.
