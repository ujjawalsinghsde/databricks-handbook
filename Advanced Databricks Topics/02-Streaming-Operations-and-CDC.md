# Streaming Operations and CDC

## Production streaming is more than a streaming read

A production stream needs a source, a checkpoint, a target, and a plan for restarts and late records. It also needs monitoring so someone can tell whether it is keeping up. For a file stream, Auto Loader often handles discovery. For Kafka or another event source, Structured Streaming can read records continuously or in repeated micro-batches.

If continuous processing is not required, an available-now run can process all data that has arrived and then stop. A scheduled job can start it again later. This pattern often gives simpler operations while still processing incrementally.

## Managed ingestion with Lakeflow Connect

Lakeflow Connect provides managed connectors for supported databases and SaaS applications. It can handle parts of the source connection, incremental ingestion, and pipeline operation for you. The exact connector list and behavior depend on the cloud and source.

I consider it when the source is supported and I want Databricks to manage more of the ingestion work. I still need to decide where the data lands, what permissions the connector needs, how updates and deletes should be handled, and how to monitor freshness. A connector gets data into Databricks; it does not automatically decide all the business rules for Silver and Gold tables.

Before choosing a managed connector, check its supported objects, initial load behavior, incremental cursor or CDC support, delete handling, schema-change behavior, limits, and cost. If it does not meet the source requirements, use the source's CDC mechanism, Auto Loader, Structured Streaming, or another ingestion tool.

## Checkpoints and recovery

A checkpoint stores a query's progress and, for stateful queries, information the query needs to continue. Use a durable location that the job identity can access. Give each streaming query its own checkpoint path. Do not share a checkpoint between different queries, even if they write to the same table.

If a query fails, restart it with the same source, compatible logic, and checkpoint. Spark can then resume from its saved progress. Deleting a checkpoint can make the query start over or lose its saved state; only do that as part of a planned recovery or migration. First decide how to prevent duplicate writes if data is replayed.

Keep the schema tracking location for Auto Loader too. It records schema history and is separate from the streaming checkpoint.

## Changing a streaming query

Some code changes are simple, like adding a stateless projection. Changes to stateful operations, keys, source options, or output behavior can make an old checkpoint incompatible or change the meaning of the state. Before changing those parts, decide whether to:

- Continue with the current checkpoint because the change is compatible.
- Start a new query with a new checkpoint and deliberately replay data.
- Run a backfill into a separate target, validate it, then switch consumers.

Do not point a changed query at an old checkpoint and assume it is safe. Test the restart and recovery path with a copy of representative data.

## Late data and watermarks

Event time means when something happened; processing time means when Databricks received it. A device might upload yesterday's events after it reconnects, so these times can be different.

A watermark sets how late an event can arrive while the query still keeps enough state to use it. A longer watermark can include more late data but requires the query to remember more information. Decide what to do with events that arrive outside the allowed delay: drop them with metrics, store them for correction, or rebuild the affected date.

For a stream-stream join, put a time range around the match. For example, accept a payment if it arrives within two hours of an order. Without a time bound, the stream may need to keep unmatched orders indefinitely.

## Monitor whether the stream is keeping up

Watch the rate at which data arrives and the rate at which it is processed. Also watch the age of the oldest unprocessed data, batch duration, failed batches, and state size. If arrivals keep outpacing processing, the backlog grows. Increasing compute may help, but first check if a slow transformation or a skewed key is causing the delay.

Set an alert on a business measure such as “orders are no more than 15 minutes behind,” not only on whether the query process is running.

## CDC basics

Change data capture (CDC) sends inserts, updates, and deletes from a source system. A CDC record usually has a business key, an operation type, and a sequence or timestamp that orders the changes.

The sequence matters. Suppose customer 7 changes twice. If the older change arrives last, it must not overwrite the newer value. The pipeline needs a reliable ordering column and a rule for what to do if two changes for the same key have the same sequence.

CDC processing also needs:

- A first load of the existing source data (initial hydration).
- Correct handling of inserts, updates, and deletes.
- A stable key and a way to order records.
- A saved progress point and a replay plan.
- Reconciliation to catch missing or duplicate changes.

## AUTO CDC in Lakeflow pipelines

Lakeflow pipelines provide `AUTO CDC` for change feeds and `AUTO CDC FROM SNAPSHOT` when the source only provides periodic snapshots. These can maintain SCD Type 1 or Type 2 tables.

- **Type 1** keeps the latest value. Old values are overwritten.
- **Type 2** keeps history. A change closes the old version and adds a new version with an effective time range.

For Type 2, use business time if reports need to know when a value was true in the source domain. Use system time if you also need to know when Databricks received or processed the change. Bitemporal support availability can vary; check current docs before using it.

The sequencing field must represent the correct order of changes. For deletes, configure which operation marks a row as deleted. For partial updates, decide whether missing/null fields mean “leave unchanged” or “set this field to null.” Those are different meanings.

## Delta Change Data Feed

Delta Change Data Feed (CDF) records row changes made to a Delta table. It is useful when a downstream table needs to process only the changes from an upstream Delta table. CDF must be enabled and its change history is subject to retention. A consumer should record its last processed version and detect if that version is no longer available.

CDF and source CDC are related but not the same. CDF exposes changes in a Delta table. Source CDC brings changes from an external database or system into Databricks.

## References

- [Auto Loader](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/)
- [Structured Streaming](https://docs.databricks.com/aws/en/structured-streaming/)
- [AUTO CDC APIs](https://docs.databricks.com/aws/en/ldp/cdc)
- [CDC and snapshots](https://docs.databricks.com/aws/en/ldp/what-is-change-data-capture)
- [Delta Change Data Feed](https://docs.databricks.com/aws/en/delta/delta-change-data-feed)
