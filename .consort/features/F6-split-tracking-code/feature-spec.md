# F6 — Split the combined tracking code into batch and serial columns

## Summary
Replace the opaque hyphen-delimited `tracking_code` with first-class `batch_number` and `serial_number` fields, delivered as expand-then-contract. Every sprint-1 row survives; nonconforming codes stay NULL and are counted by an integrity probe; the feature ends with `tracking_code` dropped.

## Data semantics
- A tracking_code "conforms" iff splitting on '-' yields exactly three non-empty segments; segment 2 → batch_number, segment 3 → serial_number. Anything else (fewer/more segments, empty segment, NULL code) is nonconforming: batch_number and serial_number are NULL.
- `location` is canonical; the code's leading segment never rewrites it.
- A NULL batch_number or serial_number is "not tracked" for that field (rendered per column, never blank).

## Stories
- S1-expand-split-columns — add and backfill batch_number/serial_number with an integrity probe while tracking_code still exists.
- S2-show-batch-serial — show Batch and Serial on the SKU detail view and as home-table columns, replacing the tracking-code column.
- S3-contract-drop-tracking-code — drop tracking_code; reversible via round-trip; end state is up-migrated.

### S1-expand-split-columns
ACs: AC1 conforming codes backfilled by delimiter; AC2 nonconforming/absent codes left NULL; AC3 every row survives with sku/location/quantity unchanged; AC4 integrity probe reports nonconforming count.

## Out of scope
- Recreating a location column (already canonical).
- Reversibility as its own story or final state.
- Validating/enforcing batch or serial formats on new input beyond the split.

## Open questions
- Codes with more than three segments (e.g. 'A-B-C-D'): recommended — treat as nonconforming (NULL, counted by probe) rather than joining extra segments.
- Should the probe block acceptance or only warn? Recommended — warn with the count; PO accepts at review (per request: "surfaces the count … for review").
