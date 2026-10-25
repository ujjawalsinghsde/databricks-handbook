# Lakeflow Pipelines and Jobs in Production

## Lakeflow Pipelines and SDP

Apache Spark Declarative Pipelines (SDP) lets you define the datasets you want and the queries that create them. You do not manually call each transformation in order. The pipeline engine sees that one dataset reads another and works out the dependency order.

Databricks Lakeflow pipelines are built on SDP. Lakeflow adds Databricks features such as managed updates, data quality expectations, AUTO CDC, and a queryable event log. This distinction is useful: the declarative authoring model is SDP; the managed Databricks service and its extra features are Lakeflow pipelines. Existing notes may call them Delta Live Tables or DLT.

### When a declarative pipeline helps

Use a pipeline when the main work is creating and refreshing related tables or views. The engine can plan dependencies and incremental updates for you. A regular Lakeflow Job is a better fit for general steps such as calling an API, copying a file, or coordinating work across several unrelated systems.

### Streaming table or materialized view?

A **streaming table** is a good fit when new records should be processed incrementally. It keeps progress and adds new input as it arrives.

A **materialized view** stores the result of a query and refreshes it as its sources change. It is useful for repeated queries or summaries when the stored result makes reads faster.

The choice depends on the update behavior you need. A streaming table is not simply a faster view, and a materialized view is not necessarily a full recompute on every update. Check the pipeline's refresh behavior and supported source types.

### Expectations and bad records

An expectation is a data rule attached to a pipeline dataset. For each rule, decide whether to just measure bad rows, drop them, or fail the update. For example:

```sql
CREATE OR REFRESH STREAMING TABLE valid_orders (
  CONSTRAINT order_id_present EXPECT (order_id IS NOT NULL) ON VIOLATION FAIL UPDATE,
  CONSTRAINT amount_not_negative EXPECT (amount >= 0) ON VIOLATION DROP ROW
)
AS SELECT * FROM STREAM(main.bronze.orders);
```

Here a missing order ID fails the update because the record cannot be identified. A negative amount is dropped. In a business pipeline, I may instead write rejected rows to a quarantine table so the source owner can fix them. If I drop rows, I still need metrics to see how many were dropped and why.

### Run modes and updates

Development mode is useful while changing code because updates are easier to inspect and iterate on. Production mode is for scheduled or automated runs. Keep the pipeline definition in Git, use a production identity to run it, and set the target catalog and schema deliberately.

An update can process new data incrementally or refresh more data, depending on the dataset and operation. Before a full refresh, check how much source data will be reread, whether upstream files are still available, and whether downstream tables will be affected.

Use the pipeline event log to inspect update progress, data quality metrics, and errors. If an update fails, identify the failing dataset and source range before retrying. A retry is safe only if the writes and downstream effects are safe to repeat.

## Lakeflow Jobs

A Lakeflow Job is a graph of tasks. A task can run a notebook, Python file, SQL statement, pipeline, or another supported task. An arrow between tasks means the downstream task waits for the upstream task to finish successfully.

For example:

```text
ingest_orders ──> clean_orders ──> build_sales_summary
                         └──────> update_customer_dimension
```

The final two tasks can run in parallel if they do not depend on each other. Keeping tasks separate makes it clear which part failed and which outputs may need rebuilding.

### Settings to think through

- **Run as:** use a service principal for production so the job is not tied to an employee account.
- **Compute:** choose job compute or serverless where it fits; set a supported runtime and required libraries.
- **Retries:** retry transient failures, not bad input or broken code. Add a reasonable timeout.
- **Maximum concurrent runs:** decide if a second run can safely start while the first one is still running.
- **Notifications:** notify the person or team that can act on the failure.
- **Parameters:** pass dates, source ranges, or environment-specific values into the job instead of hardcoding them.

### Repairing a failed run

Repair runs let you rerun failed or selected tasks in a job run. Before repairing, check whether successful upstream tasks still produced valid output. If the code or input changed, rerun downstream work that depends on it too. Do not assume a repair reruns every task or rebuilds every table.

### Backfilling old dates

A backfill reruns the regular job for earlier time ranges. Make the date range a job parameter so normal runs and backfills use the same code. Before a backfill:

1. Decide whether the target data should be replaced, merged, or appended.
2. Check that rerunning the same date will not create duplicates.
3. Estimate how much input and compute the range needs.
4. Limit concurrent runs if they could overload the source or compete with daily production jobs.
5. Check row counts and key totals after the backfill.

For Lakeflow pipelines, Databricks documents a pipeline-specific append-once pattern for historical data. Use the documented approach for the pipeline type rather than assuming a normal job backfill behaves the same way.

### Retries and idempotency

A task is idempotent when running it twice with the same input leaves the same correct result as running it once. This matters because a task can write data and then fail before reporting success. The retry may run again.

Common patterns include replacing a bounded date range, merging by a stable key, or writing to a temporary target and publishing only after checks pass. Choose based on the data and recovery requirements.

## References

- [Apache Spark Declarative Pipelines and Lakeflow](https://docs.databricks.com/aws/en/ldp/concepts/spark-declarative-pipelines)
- [Lakeflow pipeline best practices](https://docs.databricks.com/aws/en/ldp/best-practices)
- [Configure and repair Lakeflow Jobs](https://docs.databricks.com/aws/en/jobs/configure-job)
- [Backfill jobs](https://docs.databricks.com/aws/en/jobs/backfill-jobs)
- [Backfilling historical data with pipelines](https://docs.databricks.com/aws/en/ldp/flows)
