# Databricks Performance Optimization — Interview Notes

## 1. How Do You Optimize Performance in Databricks?

**Interview answer:**

I optimize Databricks performance by identifying bottlenecks, optimizing business logic, reducing unnecessary shuffles, choosing the right join strategy, managing data partitions, and optimizing Delta tables. I also use Spark UI and query execution plans to identify slow operations and measure improvements.

### Main Optimization Techniques

1. Optimize business logic.
2. Filter data early.
3. Select only required columns.
4. Avoid unnecessary transformations.
5. Optimize joins.
6. Reduce unnecessary shuffles.
7. Handle data skew.
8. Optimize partitioning.
9. Use Adaptive Query Execution (AQE).
10. Optimize Delta tables.
11. Avoid unnecessary caching.
12. Use appropriate compute resources.
13. Process data incrementally.
14. Monitor performance using Spark UI.

---

## 2. Business Logic Optimization

Business logic optimization means improving the code that processes data so that it performs less unnecessary work.

### 2.1 Filter Data Early

**Problem:** Processing millions of records when only a small subset is required.

**Solution:** Apply filters as early as possible.

```python
from pyspark.sql.functions import col

df = spark.table("orders")

filtered_df = df.filter(col("status") == "COMPLETED")
```

**Benefit:** Reduces the number of records processed by subsequent operations.

**Interview answer:** I apply filters early to reduce the volume of data processed by downstream transformations.

### 2.2 Select Only Required Columns

**Problem:** Reading unnecessary columns increases data processing and memory usage.

**Solution:**

```python
df = spark.table("orders").select(
    "order_id",
    "customer_id",
    "amount"
)
```

**Benefit:** Reduces unnecessary data reading and processing. Columnar formats such as Parquet and Delta can benefit from column pruning.

**Interview answer:** I select only the required columns to reduce I/O and improve processing efficiency.

### 2.3 Avoid Unnecessary Transformations

**Problem:** Repeated transformations increase execution overhead.

**Solution:**

```python
# Avoid repeating the same calculation
df = df.withColumn("total", col("price") * col("quantity"))
df = df.withColumn("total_after_tax", col("total") * 1.18)
```

Calculate reusable expressions once when appropriate.

**Benefit:** Avoids duplicated work and makes the code easier to maintain.

**Interview answer:** I remove redundant transformations and reuse calculated results wherever practical.

### 2.4 Prefer Built-in Functions Over Python UDFs

A UDF is a user-defined function used to apply custom logic to data.

**Problem:** Regular Python UDFs can introduce serialization overhead and limit Spark's ability to optimize expressions.

**Solution:** Prefer Spark SQL functions when they can achieve the same result.

```python
from pyspark.sql.functions import upper

df = df.withColumn("customer_name", upper(col("customer_name")))
```

Instead of writing a Python UDF just to convert text to uppercase, use the built-in `upper()` function.

**Benefit:** Built-in functions can be optimized by Spark's execution engine.

**Interview answer:** I prefer built-in Spark functions over Python UDFs because they generally allow better optimization and reduce overhead.

### 2.5 Handle Null Values Properly

**Problem:** Null values can cause incorrect calculations or unexpected results.

**Solution:**

```python
df = df.fillna({"amount": 0})
```

Use this only when replacing a missing amount with zero is correct for the business requirement.

**Benefit:** Improves data quality and avoids unintended results.

**Interview answer:** I handle null values according to business rules to ensure accurate and reliable processing.

### 2.6 Avoid Unnecessary `distinct()`

**Problem:** `distinct()` often requires a shuffle to remove duplicate rows.

```python
df = df.distinct()
```

**Solution:** Use it only when duplicate removal is required. If the requirement concerns a particular key, consider whether `dropDuplicates(["order_id"])` matches the intended logic.

**Important:** Both operations can require significant processing. Choose the operation based on the business requirement.

**Interview answer:** I avoid unnecessary deduplication because it can introduce expensive shuffle operations.

### 2.7 Avoid Repeated Calculations

**Problem:** The same expensive calculation is repeated across multiple operations.

**Solution:** Reuse a calculated DataFrame or expression where appropriate. If an expensive result is reused multiple times, consider caching it after checking memory and performance trade-offs.

**Interview answer:** I identify repeated calculations and reuse intermediate results when doing so reduces total execution time.

---

## 3. Spark Execution Optimization

### 3.1 Understand Transformations and Actions

**Transformations** define operations on data.

Examples:
- `filter()`
- `select()`
- `withColumn()`
- `join()`

**Actions** trigger execution and return results or write output.

Examples:
- `count()`
- `collect()`
- `show()`
- Writing a DataFrame to storage

Spark generally evaluates transformations lazily and builds an execution plan before executing the required work.

**Interview answer:** I use Spark's lazy evaluation and execution planning to understand how operations are combined and executed.

### 3.2 Reduce Shuffles

A shuffle redistributes data across Spark partitions, often involving network communication and disk I/O.

Operations that can trigger shuffles include:
- `groupBy()`
- `distinct()`
- Many joins
- `orderBy()`
- `repartition()`

**Solution:** Avoid unnecessary shuffles, filter data before expensive operations when appropriate, and choose suitable join and partitioning strategies.

**Interview answer:** I reduce unnecessary shuffles because they increase network communication and can significantly affect Spark job performance.

### 3.3 Repartition vs. Coalesce

**Repartition:**

```python
df = df.repartition(20)
```

- Can increase or decrease the number of partitions.
- Redistributes data.
- Usually involves a shuffle.

**Coalesce:**

```python
df = df.coalesce(5)
```

- Commonly reduces the number of partitions.
- Usually avoids a full shuffle.
- May produce uneven partitions if used without considering data distribution.

**Interview answer:** I use `repartition()` when I need to redistribute data or increase parallelism, and `coalesce()` when I need to reduce partitions without a full shuffle.

### 3.4 Choose the Right Number of Partitions

Too few partitions can limit parallelism. Too many small partitions can increase scheduling overhead.

**Solution:** Check data volume, task duration, cluster resources, and Spark UI metrics before changing partition counts.

There is no single ideal partition count for every workload.

**Interview answer:** I tune partition counts according to data size, available compute resources, and task-level performance.

### 3.5 Use Adaptive Query Execution (AQE)

AQE allows Spark to adjust certain execution decisions at runtime using statistics collected during execution.

It can help with:
- Coalescing small shuffle partitions.
- Adapting join strategies.
- Handling certain skewed joins.

**Interview answer:** I use AQE to let Spark adapt execution plans at runtime based on actual data statistics.

### 3.6 Avoid Unnecessary Global Sorting

**Problem:** Global sorting can require expensive data redistribution.

**Solution:** Use `orderBy()` only when global ordering is required. If sorting within partitions is sufficient, consider `sortWithinPartitions()`.

**Interview answer:** I avoid unnecessary global sorting because it can introduce expensive shuffles.

### 3.7 Optimize Window Functions

Window functions are useful for rankings, running totals, and comparisons between related rows.

**Problem:** Large window operations can require repartitioning and sorting.

**Solution:**
- Filter unnecessary rows before the window operation.
- Partition by the correct business key.
- Avoid unnecessary window calculations.
- Use an appropriate ordering column.

**Interview answer:** I optimize window functions by reducing input data and using the correct partitioning and ordering columns.

---

## 4. Join Optimization

### 4.1 Broadcast Join

Use a broadcast join when one DataFrame is small enough to distribute efficiently to the executors.

```python
from pyspark.sql.functions import broadcast

result = large_df.join(
    broadcast(small_df),
    "customer_id",
    "inner"
)
```

**Benefit:** Can avoid shuffling the large DataFrame for the join.

**Important:** Broadcasting an excessively large DataFrame can cause memory problems.

**Interview answer:** I use broadcast joins when one side is sufficiently small to avoid unnecessary shuffling of the large dataset.

### 4.2 Sort-Merge Join

A sort-merge join is commonly used for large DataFrames when both sides are too large to broadcast.

Spark may shuffle and sort the data before joining.

**Optimization:** Reduce input size, select required columns, and filter data early.

**Interview answer:** For large datasets, Spark may use sort-merge joins. I optimize them by reducing data volume and avoiding unnecessary shuffle work.

### 4.3 Handle Data Skew

Data skew occurs when some partitions contain much more data than others.

**Example:** One customer has millions of transactions, while most customers have only a few hundred.

**Symptoms:**
- Most tasks finish quickly.
- A few tasks take much longer.
- Some executors remain busy after others finish.

**Solutions:**
- Enable or use AQE skew-join optimization where appropriate.
- Filter unnecessary records.
- Broadcast the smaller side if suitable.
- Use salting for certain skewed joins.
- Reconsider partitioning and data distribution.

**Interview answer:** I identify skew using Spark UI and address it with AQE, suitable join strategies, or salting when necessary.

### 4.4 What Is Salting?

Salting adds an additional value to a skewed key to distribute its data across multiple partitions.

For example, a heavily skewed customer key can be expanded into several salted keys and matched with corresponding keys on the other side of a join.

**Benefit:** Can distribute the work more evenly.

**Important:** Salting adds complexity and should be used only when skew is confirmed and simpler solutions are insufficient.

---

## 5. Partitioning and File Optimization

### 5.1 Partition Pruning

Partition pruning means reading only the table partitions needed by a query.

For example, if a table is partitioned by `order_date`, a query filtering on a particular date may read fewer partitions.

```sql
SELECT *
FROM orders
WHERE order_date = '2026-10-25';
```

**Benefit:** Reduces the amount of data read when the filter matches the table's partitioning scheme.

**Interview answer:** I use appropriate partitioning and filters to enable partition pruning and reduce unnecessary data scanning.

### 5.2 Avoid Too Many Small Files

**Problem:** A table containing thousands of tiny files can incur additional file-listing and task-scheduling overhead.

**Solution:** Compact small files where appropriate.

For Delta tables:

```sql
OPTIMIZE catalog_name.schema_name.orders;
```

**Benefit:** Combines smaller files into more efficient data files.

### 5.3 Avoid Oversized Files and Poor Partitioning

Large files are not automatically bad, but an unsuitable file layout can limit parallelism or cause inefficient scans.

Choose a layout based on:
- Table size.
- Query patterns.
- Data distribution.
- Write frequency.
- File sizes and task metrics.

**Interview answer:** I optimize file sizes and partitioning based on actual query patterns rather than applying one fixed layout to every table.

---

## 6. Cache, Persist, and Checkpoint

### 6.1 Cache

Caching stores a DataFrame for potential reuse.

```python
df.cache()
df.count()
```

The action can materialize the cached data.

**Use it when:** An expensive DataFrame is reused multiple times.

**Avoid it when:** The data is used only once or caching causes memory pressure.

### 6.2 Persist

`persist()` allows you to choose a storage level.

```python
from pyspark import StorageLevel

df.persist(StorageLevel.MEMORY_AND_DISK)
```

This storage level allows cached data to use memory and spill to disk when necessary.

### 6.3 Checkpoint

Checkpointing saves a DataFrame to reliable storage and truncates its existing lineage.

It can help when a processing plan becomes excessively long or when a reliable recovery boundary is needed.

### 6.4 Local Checkpoint

A local checkpoint can truncate lineage using executor-local storage, but it is less reliable than a durable checkpoint.

**Interview answer:** I use caching for repeated computations and checkpointing when I need to truncate long lineage. I avoid both when their overhead outweighs their benefits.

---

## 7. Delta Lake Optimization

### 7.1 OPTIMIZE

`OPTIMIZE` compacts Delta table files.

```sql
OPTIMIZE catalog_name.schema_name.orders;
```

**Benefit:** Can reduce small-file overhead and improve query performance.

### 7.2 Z-ORDER

Z-ORDER reorganizes data to improve data skipping for suitable filter columns.

```sql
OPTIMIZE catalog_name.schema_name.orders
ZORDER BY (customer_id);
```

**Use it when:** Queries frequently filter on suitable columns and the table's layout benefits from Z-Ordering.

**Important:** Z-Ordering is not automatically the best choice for every table. Consider the Databricks runtime, table layout, and available alternatives.

### 7.3 Liquid Clustering

Liquid clustering organizes data around selected clustering columns and can adapt the data layout as requirements evolve.

It is an alternative to traditional partitioning and can be preferable for supported workloads.

**Interview answer:** I evaluate liquid clustering and Z-Ordering based on the table's query patterns and supported Databricks features.

### 7.4 Data Skipping

Delta Lake collects statistics for supported columns and can skip files that cannot contain matching records.

**Benefit:** Reduces unnecessary file reads.

**Interview answer:** I improve data skipping through suitable data layout and filter patterns so that queries read fewer files.

### 7.5 VACUUM

`VACUUM` removes eligible obsolete data files that are no longer needed by the table's retained history and retention rules.

```sql
VACUUM catalog_name.schema_name.orders;
```

**Important:** `VACUUM` is primarily a storage cleanup operation, not a general query-performance optimization. Changing retention settings can affect time travel and recovery, so follow the appropriate safety checks.

### 7.6 Optimize MERGE Operations

`MERGE` is used to insert, update, or delete matching records.

**Optimization techniques:**
- Filter the source data before merging.
- Use suitable join keys.
- Process incremental changes rather than the entire source when possible.
- Avoid scanning unnecessary target data.
- Monitor files rewritten and data scanned.

**Interview answer:** I optimize Delta MERGE operations by reducing source data, limiting unnecessary target scans, and processing only the required changes.

### 7.7 Change Data Feed (CDF)

CDF records row-level changes made to a Delta table when enabled and supported.

It can help downstream pipelines process incremental changes instead of rereading an entire table.

**Interview answer:** I use CDF when appropriate to identify changed records and reduce unnecessary full-table processing.

---

## 8. Databricks Compute Optimization

### 8.1 Choose the Right Cluster Size

Too little compute can make a workload slow. Too much compute can increase cost without a proportional performance improvement.

Evaluate:
- CPU utilization.
- Memory pressure.
- Executor failures.
- Task duration.
- Shuffle volume.
- Data volume.

**Interview answer:** I size compute based on workload metrics and balance execution time against cost.

### 8.2 Autoscaling

Autoscaling adjusts the number of workers within configured limits based on workload demand.

It can improve resource utilization, but it does not automatically fix skew or inefficient business logic.

### 8.3 Photon

Photon is Databricks' vectorized execution engine for supported workloads.

It can improve performance for suitable SQL and DataFrame operations.

**Interview answer:** I evaluate Photon for supported workloads and compare execution time and cost before and after enabling it.

### 8.4 Jobs Compute vs. All-Purpose Compute

Jobs compute is intended for scheduled or automated workloads. All-purpose compute is commonly used for interactive development and exploration.

**Interview answer:** I choose the compute type based on workload requirements, operational practices, and cost.

### 8.5 SQL Warehouse

A SQL warehouse is compute designed for SQL analytics and supported BI workloads.

Use an appropriately configured warehouse for SQL queries and dashboards rather than assuming every workload should run on the same compute resource.

---

## 9. ETL Pipeline Optimization

### 9.1 Incremental Processing

**Problem:** Processing the entire dataset every time wastes compute when only a small amount of data changes.

**Solution:** Process only new or changed records using suitable mechanisms such as:
- Watermarks.
- Change Data Feed.
- Source change tracking.
- Reliable incremental ingestion patterns.

**Interview answer:** I use incremental processing to avoid reprocessing unchanged data and reduce pipeline execution time and cost.

### 9.2 Predicate Pushdown

Predicate pushdown allows supported filters to be applied close to the data source or storage reader.

**Benefit:** Can reduce the amount of data read.

**Interview answer:** I use supported filter conditions and inspect execution plans to confirm that unnecessary data reading is reduced.

### 9.3 Use Columnar Formats

Formats such as Parquet and Delta support column-oriented reading and can improve analytical workloads.

**Benefit:** Queries can read selected columns instead of loading every column.

### 9.4 Make Pipelines Idempotent

An idempotent pipeline can be rerun without unintentionally duplicating or corrupting the result.

**Techniques:**
- Use stable business keys.
- Use suitable `MERGE` logic.
- Track processing state.
- Design retry-safe writes.
- Validate source and target counts where appropriate.

**Interview answer:** I design pipelines to be idempotent so that retries do not create duplicate records or inconsistent results.

### 9.5 Data Validation and Reconciliation

During migration from Redshift to Databricks, compare:
- Record counts.
- Aggregated totals.
- Key-level results.
- Null counts.
- Duplicate counts.
- Business-critical values.

**Interview answer:** I validate data after each important stage to ensure that performance improvements do not change business results.

---

## 10. How Do You Identify a Performance Bottleneck?

Do not randomly change configurations. First identify the bottleneck.

### Step 1: Check Spark UI

Look for:
- Long-running stages.
- Slow tasks.
- Shuffle read and write.
- Spill to disk.
- Uneven task duration.
- Failed tasks.
- Executor memory pressure.

### Step 2: Inspect the Execution Plan

```python
df.explain("formatted")
```

Check:
- Join strategy.
- Exchanges indicating shuffle boundaries.
- Filters.
- Scans.
- Sort operations.
- Other expensive operators.

### Step 3: Check the Data Layout

Inspect:
- Number of files.
- File sizes.
- Partition distribution.
- Table statistics.
- Whether filters can skip files.

### Step 4: Change One Important Thing at a Time

For example:
- Apply filtering earlier.
- Change the join strategy.
- Reduce unnecessary shuffles.
- Optimize small files.
- Adjust compute resources.

### Step 5: Compare Before and After

Measure:
- Total execution time.
- Data scanned.
- Shuffle volume.
- Number of files read.
- Compute usage and cost.
- Output correctness.

**Interview answer:** I use Spark UI, execution plans, and table metrics to identify the bottleneck. I then apply a targeted optimization and compare execution time, resource usage, cost, and output correctness.

---

## 11. Common Interview Questions and Short Answers

### Q1. How do you optimize a slow Spark job?

I identify the bottleneck using Spark UI and execution plans, then optimize the relevant logic, joins, shuffles, partitions, or compute resources.

### Q2. Why are shuffles expensive?

Shuffles redistribute data across partitions and may require network communication, disk I/O, and serialization.

### Q3. What is data skew?

Data skew occurs when some partitions contain significantly more data than others, causing a few tasks to run much longer.

### Q4. How do you handle data skew?

I inspect task durations and data distribution, then consider AQE, broadcast joins, salting, or better partitioning.

### Q5. What is a broadcast join?

A broadcast join distributes a sufficiently small dataset to the executors so that the large dataset may not need to be shuffled for the join.

### Q6. What is AQE?

Adaptive Query Execution adjusts supported parts of Spark's execution plan at runtime based on actual data statistics.

### Q7. What is partition pruning?

Partition pruning skips irrelevant table partitions when a query filters on suitable partition columns.

### Q8. What is the difference between repartition and coalesce?

`repartition()` redistributes data and usually causes a shuffle. `coalesce()` commonly reduces partitions without a full shuffle.

### Q9. When should you cache a DataFrame?

When an expensive DataFrame is reused multiple times and caching improves overall execution time without causing excessive memory pressure.

### Q10. What is OPTIMIZE in Delta Lake?

It compacts Delta table files to improve file layout and can reduce small-file overhead.

### Q11. What is Z-Ordering?

Z-Ordering reorganizes data to improve data skipping for suitable filter columns.

### Q12. What is the difference between cache and checkpoint?

Cache stores computed data for reuse. Checkpointing saves data to a checkpoint location and truncates lineage.

### Q13. How do you optimize a large MERGE?

I reduce source data, use suitable keys, limit unnecessary target scans, and process incremental changes where possible.

### Q14. Why should we avoid Python UDFs when possible?

Regular Python UDFs can introduce serialization overhead and may prevent some Spark SQL optimizations. Built-in functions are preferable when they meet the requirement.

### Q15. How do you prove that your optimization worked?

I compare execution time, data scanned, shuffle volume, compute cost, and output correctness before and after the change.

---

## 12. Practical Example: Redshift to Databricks Migration

Suppose we are migrating a large table from Amazon Redshift to Databricks.

### Initial Problem

- The source table contains billions of records.
- Data ingestion takes a long time.
- Some Spark stages have long-running tasks.
- The target Delta table contains many small files.
- Reprocessing the complete dataset increases cost.

### Optimization Approach

**1. Improve ingestion**

Unload the required source data to Amazon S3 in an appropriate columnar format, such as Parquet, when suitable for the migration design.

**2. Reduce unnecessary data**

Read only the required columns and filter data when the migration requirements permit it.

**3. Optimize transformations**

Use built-in Spark functions and remove unnecessary transformations.

**4. Optimize joins**

Broadcast small reference datasets where appropriate and investigate skewed joins.

**5. Improve file layout**

Compact small Delta files when the table's workload benefits from compaction.

**6. Process incrementally**

After the initial load, process only new or changed records where the source and pipeline design support reliable incremental processing.

**7. Validate the output**

Compare counts, key aggregates, and business-critical values between the source and target.

**8. Measure performance**

Compare the execution time, data scanned, shuffle volume, and compute cost before and after optimization.

### Interview Answer

In my Redshift-to-Databricks migration project, I focus on optimizing the complete pipeline rather than only changing Spark configurations. I reduce unnecessary data processing, optimize transformations and joins, manage Delta file sizes, and use incremental processing where possible. I validate the migrated data and compare performance metrics to confirm that the changes improve efficiency without affecting correctness.

---

## 13. Quick Revision Checklist

Before running a Spark job, ask:

- Am I reading only the required columns?
- Can I filter unnecessary records earlier?
- Am I using built-in functions instead of unnecessary Python UDFs?
- Am I performing unnecessary `distinct()`, sorting, or transformations?
- Can I reduce shuffle operations?
- Is one side of a join small enough to broadcast?
- Is data skew causing slow tasks?
- Is the partition count appropriate?
- Can AQE improve the execution plan?
- Are too many small files slowing down reads?
- Would Delta `OPTIMIZE` benefit this workload?
- Are Z-Ordering or liquid clustering appropriate for the table?
- Am I processing only changed data when possible?
- Is caching justified by repeated use?
- Have I checked Spark UI and `explain()`?
- Have I compared performance, cost, and output correctness?

---

## 14. One-Minute Interview Summary

I optimize Databricks performance by first identifying bottlenecks using Spark UI and execution plans. I then reduce unnecessary data processing by filtering early, selecting only required columns, and using built-in Spark functions. I optimize joins using broadcast joins where appropriate and handle data skew with AQE or salting when needed. I also reduce unnecessary shuffles, tune partitions, compact Delta files, and evaluate Z-Ordering or liquid clustering based on query patterns. For ETL pipelines, I use incremental processing where possible and validate the output. Finally, I compare execution time, resource usage, cost, and data correctness to confirm that the optimization is effective.