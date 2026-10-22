# StockFlow – parallel snapshot: F2 built + F6 designed

## What this is

A **parallel-development snapshot** of StockFlow, a warehouse-management app for a mid-market
operation that has outgrown spreadsheets. It captures two features worked on **concurrently**, each
on its own Lakebase branch forked from the shared trunk:

- **`feature-f2-stock-adjustment` — F2 taken all the way to BUILD-complete.** "Adjust the stock
  level of a SKU at a location": all three stories built, tests GREEN against F2's Lakebase branch.
- **`feature-f6-split-tracking-code` — F6 taken to DESIGN-complete.** "Split the tracking code into
  batch + serial columns": the full design lane authored and gated, ready to build.

Neither branch contains the other's work — that's the point. They forked from the same trunk and
advanced in parallel, exactly as two developers (or two agents) would on branched databases: one
building F2, the other designing F6, without blocking each other.

It's built with **[Consort](https://github.com/databricks-solutions/consort)**, a spec-first,
test-driven agent framework for apps on **Lakebase** (Databricks serverless Postgres). Consort walks
a feature through a *design* lane (plan → spec → UX → architecture → tests) and a *build* lane (TDD:
RED → GREEN → REFACTOR). This snapshot shows one feature mid-way through the second lane and another
at the end of the first — at the same time.

## Where we are

- **Feature F1 – "Record and view stock by SKU and location":** built (the trunk — the React SPA +
  JSON API that files, retrieves, and lists a SKU's stock by location).
- **Feature F2 – "Adjust the stock level of a SKU at a location":** **build complete** on
  `feature-f2-stock-adjustment`, with its build turn-cycle (event log + turns + per-test table
  snapshots) captured for dashboard replay.
  - `S1-adjust-to-value` · `S2-adjust-by-delta` · `S3-reject-negative-stock` — all built, tests green.
- **Feature F6 – "Split the tracking code":** **design complete** on `feature-f6-split-tracking-code`
  (feature-spec, UX design-guide, architecture, db-design, ordered test-list — authored and gated).
  - `S1-expand-split-columns` · `S2-show-batch-serial` · `S3-contract-drop-tracking-code` — ready to build.

The full combined run (F1 + F2 build + F6 design) replays end-to-end in the Consort dashboard; use
the feature switcher to move between F2's build and F6's design.

## How to use it

**Prerequisite:** Consort is installed, your environment is prepped, and access to a Lakebase
profile is ready to go. If you haven't set that up yet, run:

```bash
bash <(curl -sL https://tinyurl.com/consort-bootstrap)
```

If you need Lakebase, you can try it on [Free Edition](https://login.databricks.com/signup?intent=SIGN_UP&provider=DB_FREE_TIER).

### 1. Clone it down

```bash
git clone https://github.com/kevin-hartman/stockflow-example-parallel-f2-build-f6-design-complete.git
cd stockflow-example-parallel-f2-build-f6-design-complete
git checkout feature-f6-split-tracking-code
(cd client && npm install)
```

Both feature branches come down with the clone — `feature-f2-stock-adjustment` (F2 built) and
`feature-f6-split-tracking-code` (F6 designed, ready to build). Checking out `feature-f6` makes it
the branch `setup.sh` leaves you on to build; `setup.sh` recreates the paired Lakebase branch for
**both** features regardless of which one you're on.

### 2. Stand it up as *your* project

Run the setup script, replacing `my-stockflow` with your own project name:

```bash
./setup.sh my-stockflow
```

Setup recreates the parallel Lakebase branches and leaves you on `feature-f6-split-tracking-code`,
ready to build F6. If you encounter problems, read the [runbook](RUNBOOK-rehydrate-as-new-project.md)
and share it with a Claude session to resolve.

### 3. Build F6, inspect F2, or replay both

In your Claude session, start the workflow:

```
/consort:start
```

From `feature-f6-split-tracking-code` it picks up at F6's build lane (RED → GREEN → REFACTOR against
F6's Lakebase branch). Switch to `feature-f2-stock-adjustment` to inspect F2 fully built, or open the
dashboard to replay the whole parallel run.
