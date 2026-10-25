# Deployment with Declarative Automation Bundles

## Why deployment needs more than copying notebooks

A notebook in a workspace is only one part of a production pipeline. The job also has a schedule, task dependencies, compute settings, permissions, parameters, and sometimes a Lakeflow pipeline. If those settings only exist in the workspace UI, it is hard to review or recreate the setup.

Declarative Automation Bundles (formerly Databricks Asset Bundles) let me keep source code and Databricks resource definitions together in Git. The CLI can validate and deploy them. The command is still `databricks bundle`.

## What goes in a bundle

A project bundle commonly contains:

- Python or SQL code.
- Job definitions and task settings.
- Lakeflow pipeline definitions.
- Environment targets such as dev, test, and prod.
- Permissions and run identities.
- Tests and deployment scripts.

For infrastructure outside the Databricks project, such as cloud networking or workspace creation, Terraform is often a better fit. Bundles and Terraform can work together.

## A simple project shape

```text
orders/
  src/
  tests/
  resources/
    orders_job.yml
  databricks.yml
```

The job definition lives in `resources`; the main `databricks.yml` connects resources to deployment targets. A simplified configuration might look like this:

```yaml
bundle:
  name: orders

include:
  - resources/*.yml

targets:
  dev:
    mode: development
    workspace:
      host: ${var.dev_workspace}
  prod:
    mode: production
    workspace:
      host: ${var.prod_workspace}
```

This sketch leaves out the job resource and authentication details. In a real bundle, define those in resource files and provide workspace hosts and identity information through secure CI configuration or supported authentication. Do not commit tokens in YAML.

## Deployment flow

1. Work on a branch and open a pull request.
2. Run unit tests and validate the bundle.
3. Review both code and resource changes.
4. Deploy to a development or test workspace.
5. Run the job against test data and check permissions and outputs.
6. Promote the same reviewed change to production.

Useful CLI steps include:

```text
databricks bundle validate -t dev
databricks bundle deploy -t dev
databricks bundle run orders_job -t dev
```

The exact resource name and settings depend on the project. Validation catches configuration problems; it does not prove that the output data is correct, so the pipeline still needs data and integration checks.

## Environment targets

Use targets to change environment-specific values such as catalog, schema, warehouse, notifications, and run identity. Keep the transformation code the same across environments. If dev and prod use copied code folders, one environment will eventually miss a fix.

Production deployments should run under a service principal or another supported workload identity. Give that identity permission to deploy the resources and access the data the job needs. Avoid using a developer's personal token as the permanent production identity.

## Testing and rollback

Test transformations with unit tests. Then run integration checks in a non-production catalog. Check that the production identity can access its inputs and outputs before promoting the job.

For rollback, restoring the previous code is only one part. If a deployment already wrote data, decide whether to rerun corrected code, restore a table version, or rebuild the affected range. Keep the deployed commit and job run linked so you can identify exactly what changed.

## CLI and REST API

Use the CLI and bundles for repeatable deployment. Use the REST API when an integration needs an operation that the CLI does not provide. API calls should handle pagination, rate limits, and retries. A timeout can happen after the server has completed a request, so make create/update operations safe to repeat.

## References

- [Declarative Automation Bundles](https://docs.databricks.com/aws/en/dev-tools/bundles/)
- [Bundle configuration](https://docs.databricks.com/aws/en/dev-tools/bundles/settings)
- [CI/CD workflows](https://docs.databricks.com/aws/en/dev-tools/ci-cd/flows)
- [Databricks CLI](https://docs.databricks.com/aws/en/dev-tools/cli/)
