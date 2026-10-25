# Monitoring, Reliability, and Recovery

## Decide what “healthy” means

A green job run only tells me that the task finished without reporting an error. It does not prove that all expected data arrived or that the output is correct.

For each important table or pipeline, decide:

- How fresh the data should be.
- What input range or source count is expected.
- Which quality rules must pass.
- How long the job is allowed to run.
- Who owns the data and gets paged or notified.

For example: “Orders should be ready by 6:30 AM, contain all source files for the previous day, have no null order IDs, and not contain duplicate order IDs.” Those expectations can be checked and alerted on.

## What to monitor in Databricks

- **Lakeflow Jobs:** run status, task failures, duration, retries, and queueing.
- **Lakeflow pipelines:** update status, event logs, expectation results, and source progress.
- **Streaming jobs:** input rate, processing rate, lag, batch duration, and state size.
- **Databricks SQL:** query history, queue time, query profile, and warehouse usage.
- **Account usage:** system tables for billing, jobs, queries, audit activity, and other supported history.

Choose alerts that need action. An alert should say which table or job is affected, what failed, when it happened, and who owns the next step. Too many low-value alerts get ignored.

## Find the cause before retrying

When a job fails, identify the failed task and the input range it was processing. Check the error and earlier task outputs. The cause may be a source outage, changed schema, missing permission, compute limit, code error, or partial write.

Then choose the right action:

- **Retry** if the failure was transient and repeating the task is safe.
- **Repair** a job run if only failed tasks need to be rerun and earlier outputs remain valid.
- **Backfill** if a date range was missed or needs correction.
- **Replay** from a source offset or version when input needs to be read again.
- **Restore or rebuild** if the table contents are wrong.

These actions are not interchangeable. Retrying does not repair bad source records, and restoring one table may not restore every downstream table.

## Recovery point and recovery time

For critical data, agree on two numbers:

- **RPO (Recovery Point Objective):** how much recent data can be lost?
- **RTO (Recovery Time Objective):** how long can the data service be unavailable?

If the source retains CDC for 14 days and a pipeline is down for 2 days, the missed changes may be replayable. If the source only keeps 24 hours of history, a longer outage may require a backup or a full snapshot. Write down these limits before an outage.

## Disaster recovery planning

Plan for the parts separately: cloud data, Unity Catalog objects and grants, workspace resources, network access, job and pipeline definitions, and identity configuration. Keep code and Databricks resource definitions in Git. Know which tables can be rebuilt from source and which need a replica or backup.

Delta time travel can help recover a table version while its old files remain. It is not a complete backup. `VACUUM` can delete old files, and table history will not recreate a lost workspace or cloud account.

Write a runbook with the recovery steps and owners. Test it in a separate workspace or environment, then record what worked and what was missing. A recovery plan that has never been tried may not work when needed.

## References

- [Observability best practices](https://docs.databricks.com/aws/en/data-engineering/observability-best-practices)
- [System tables](https://docs.databricks.com/aws/en/admin/system-tables/)
- [Lakeflow Jobs monitoring](https://docs.databricks.com/aws/en/jobs/monitor/)
- [Lakehouse architecture deployment guide](https://docs.databricks.com/aws/en/lakehouse-architecture/deployment-guide/)
