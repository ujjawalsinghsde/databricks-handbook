# Production Design Scenarios

These are practice cases for explaining how I would design and operate a Databricks solution. In an interview or design review, I start by asking about freshness, volume, source behavior, data quality, security, and recovery. Then I explain the choices and their tradeoffs.

## Case 1: Daily files arrive late

**Situation:** A source drops order JSON files into cloud storage. Some files arrive late or are resent. The business needs a daily Gold sales table by 7 AM.

**How I would approach it:**

1. Use Auto Loader to incrementally discover files and write the raw records plus file name and ingestion time into a Bronze Delta table.
2. Configure a schema location and a durable checkpoint. Decide how new columns and malformed records should be handled.
3. In Silver, validate order IDs and amounts, deduplicate by a source event key, and preserve rejected rows for investigation.
4. Build the daily Gold result for an explicit business date. Use a retry-safe write so rerunning a date does not double-count it.
5. Measure freshness, input file count, row count, and duplicate count. Alert the owner if the data is late or looks unusual.
6. Make a backfill parameter so an operator can rebuild one date using the same logic.

The key questions are whether the source can resend files, how late files can arrive, and whether the business wants old dates corrected automatically.

## Case 2: Keep a customer table in sync with database changes

**Situation:** A source database sends customer inserts, updates, and deletes. Analysts need the current customer record, and auditors need to know what the record looked like last month.

**How I would approach it:**

1. Load the initial snapshot into a Bronze table.
2. Ingest the CDC feed and keep the operation type, business key, and sequence value.
3. Use Lakeflow AUTO CDC if the pipeline edition and source format support it.
4. Choose SCD Type 2 because the business needs history. Confirm that the sequence field correctly orders changes for the same key.
5. Apply deletes according to the source's delete event. Decide what null fields mean for partial updates.
6. Reconcile current records against the source and monitor how far behind the feed is.

If history is not needed, Type 1 may be simpler. If there is no CDC feed and only snapshots are available, compare ordered snapshots with AUTO CDC FROM SNAPSHOT where supported.

## Case 3: Dev and production must not share access

**Situation:** Engineers need a development workspace, while production data must only be available to approved production jobs and analysts.

**How I would approach it:**

1. Use Unity Catalog with a clear catalog and schema layout for dev and production.
2. Bind production catalogs to the workspaces that are allowed to use them, where supported.
3. Grant access to identity-provider groups, not individual engineers one by one.
4. Run production jobs as service principals with only the required read/write grants.
5. Store production files through governed external locations or managed tables, not ungoverned mounts.
6. Test access as a developer, analyst, and production job identity. Review audit activity and lineage.

The important check is that the development identity cannot reach production data just because it can reach a workspace attached to the metastore.

## Case 4: A dashboard query got expensive

**Situation:** A dashboard that used to load quickly now scans a large table and has long queue times.

**How I would approach it:**

1. Use query history and the query profile to separate queue time from execution time.
2. Check how many bytes and files the query reads and which joins or aggregations take time.
3. Confirm that filters are applied as expected and that the table layout fits the query pattern.
4. For managed Delta tables, check predictive optimization and consider liquid clustering on common filters.
5. Review warehouse size and concurrency if queue time is the main issue.
6. Compare latency and cost with the same query and representative data after the change.

I avoid resizing the warehouse until I know whether the issue is waiting for compute or doing too much work.

## Case 5: A stream stopped for several hours

**Situation:** An ingestion stream failed during a cloud storage outage and has a growing backlog.

**How I would approach it:**

1. Check the source availability, error, checkpoint, and last successful progress.
2. Keep the checkpoint if the query and source are still compatible. Restart and monitor lag.
3. Estimate whether normal processing can catch up before the next business deadline.
4. If the checkpoint is damaged or the query changed, create a deliberate replay plan instead of deleting state blindly.
5. Confirm that the sink is idempotent so replay does not duplicate rows.
6. Reconcile source and target counts for the outage period and report any remaining gap.

## Case 6: A schema change broke downstream tables

**Situation:** A source changed a field from a number to a string, and several downstream jobs failed.

**How I would approach it:**

1. Stop or quarantine the incompatible input so it does not silently corrupt data.
2. Identify affected tables and consumers with lineage and job history.
3. Decide the canonical type and how to handle values that cannot be converted.
4. Update the schema and transformations in a backward-compatible way if possible.
5. Test old and new records, deploy through the normal pipeline, then backfill the affected range.
6. Add a schema-change alert or contract check so the next change is caught earlier.

## A simple way to explain a design

For any case, I try to cover these points in order:

1. What freshness and correctness does the user need?
2. How does the source send data, and can it resend or delete records?
3. Which Databricks feature fits: batch job, Auto Loader, streaming, Lakeflow pipeline, or connector?
4. How will data be stored, modeled, and governed in Unity Catalog?
5. How will retries, late data, backfills, and schema changes work?
6. How will I monitor freshness, failures, quality, and cost?
7. What is the recovery plan if the source, workspace, or region is unavailable?

There is rarely one correct design without knowing the workload. The useful part is explaining why the design fits and what tradeoffs it brings.
