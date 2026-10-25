# Delta Lake Performance and Cost

## Start with the query profile

When a Databricks SQL query is slow, open its query profile and check where the time went. Look at how much data it scanned, whether it waited in a queue, and which operators took the most time. The profile can show scans, joins, aggregations, sorts, and spills. This is a good place to start before changing the warehouse size.

For Spark jobs, use the Spark UI for the execution details. General Spark tuning belongs in my Spark notes; this page focuses on Databricks features, Delta table layout, and compute choices.

## Choose a table layout from actual queries

Databricks has used partitions and `ZORDER` for table layout. Newer Delta tables can use **liquid clustering**, which organizes data around columns commonly used in filters. It is easier to adjust clustering keys as query patterns change than to redesign fixed partitions.

Do not partition a table just because it is large. A partition helps when queries filter on that column and each partition is large enough to be useful. Too many small partitions create lots of small files. For an existing partitioned table, check which queries depend on the current layout before migrating it.

## Predictive optimization and maintenance

Unity Catalog managed tables can use **predictive optimization**. Databricks can decide when to run maintenance such as `OPTIMIZE`, `VACUUM`, and statistics collection. This can remove the need for a hand-written maintenance schedule for those tables.

For tables not covered by predictive optimization, plan maintenance based on file size, query patterns, and storage retention. `OPTIMIZE` rewrites files to improve layout and reduce small-file overhead. `VACUUM` removes old unreferenced files, which may also remove the ability to time travel to versions that need them. Set the retention period with recovery requirements in mind.

## Deletion vectors and row-level changes

An update or delete can otherwise require rewriting a whole Parquet file even if only a few rows changed. **Deletion vectors** let Delta mark deleted rows without immediately rewriting the entire file. Databricks can use them to improve `DELETE`, `UPDATE`, and `MERGE` performance, especially with Photon.

Check compatibility before enabling a table feature if another engine reads the table. External Delta clients may not support every Delta feature or protocol version. A table feature can therefore improve Databricks performance while changing what older clients can read.

Row-level concurrency can reduce conflicts when concurrent writers modify different rows, but the table and runtime must support the required features. It does not mean every pair of concurrent writes is conflict-free. Keep ownership and write patterns clear.

## Photon and compute choices

Photon is Databricks' vectorized query engine for supported workloads. It can speed up SQL and DataFrame operations without changing the query. Whether it helps depends on the operation and data, so compare a representative run rather than assuming it always gives the same improvement.

Use job compute for scheduled work where that fits. Use SQL warehouses for interactive SQL and BI. Serverless can simplify operations where available. Compare runtime, concurrency, and cost for the real workload. For SQL warehouses, separate query execution time from queue time: a query waiting for capacity needs a different fix from a query with a slow plan.

## Cost attribution

Tag jobs and compute with values such as team, environment, and product. Review usage data and budgets to find idle compute, repeated failures, long-running queries, and unexpected growth. Cost changes can signal a data issue too: a sudden increase may mean a stream is stuck retrying or a query began scanning much more data.

Compare **cost per useful result**, such as dollars per million records processed. A job that finishes in half the time can still be more expensive if it uses far more compute.

## A practical example

Suppose a dashboard became slow after a table grew. The query profile shows that it reads almost every file even though the dashboard filters to one month. Check the table layout and whether the filter column is a good clustering key. If the table is managed by Unity Catalog, check whether predictive optimization is enabled. Run the same query before and after the change and compare bytes read, runtime, and cost.

## References

- [Query performance insights](https://docs.databricks.com/aws/en/sql/user/queries/performance-insights)
- [Optimization recommendations](https://docs.databricks.com/aws/en/optimizations)
- [Liquid clustering](https://docs.databricks.com/aws/en/tables/clustering)
- [Predictive optimization](https://docs.databricks.com/aws/en/optimizations/predictive-optimization)
- [Deletion vectors](https://docs.databricks.com/aws/en/delta/deletion-vectors)
- [Cost optimization](https://docs.databricks.com/aws/en/lakehouse-architecture/cost-optimization/best-practices)
