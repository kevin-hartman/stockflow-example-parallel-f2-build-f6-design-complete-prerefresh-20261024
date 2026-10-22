# F6 DB design
stock_records gains nullable `batch_number`, `serial_number` (text, no default).
S1 (SC1-add-batch-serial-columns): expand migration adds both columns and backfills from conforming tracking_code in one transaction; downgrade drops only the two columns.
- PI1: columns nullable, no default; nonconforming → NULL.
- PI2: additive add_column; downgrade drop_column only; other columns untouched.
- PI3: add + backfill within one Alembic migration transaction (Postgres transactional DDL).
- PI4: existing UNIQUE(sku,location) and ck_stock_records_quantity_non_negative retained; backfill never writes location.
No access patterns declared; no new indexes.

## S3 contract (SC2-drop-tracking-code)
Drops `tracking_code` in one transaction. The covering index `ix_stock_records_location_sku_held` INCLUDEs tracking_code, so it is rebuilt with INCLUDE (quantity, batch_number, serial_number) before the drop.
- PI5: downgrade re-adds nullable tracking_code = location-batch-serial when both are non-NULL, else NULL; no rows deleted; restores the original index.
- PI6: index rebuild + drop happen in a single Alembic transaction; if it fails, the table stays at the expand revision.
- PI7: uq_stock_records_sku_location and ck_stock_records_quantity_non_negative are left alone; batch/serial stay nullable.
Final table shape in tables[] is the post-contract one (no tracking_code).
