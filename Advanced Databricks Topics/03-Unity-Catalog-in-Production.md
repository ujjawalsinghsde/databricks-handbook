# Unity Catalog in Production

## Plan the catalog layout

Unity Catalog organizes tables and other data objects under a metastore:

```text
metastore
└── catalog
    └── schema
        └── table / view / volume / function
```

A table name such as `prod_sales.curated.orders` has a catalog, schema, and table. Some teams use catalogs for environments (`dev`, `test`, `prod`); other teams use them for business domains. Pick a pattern that matches how teams own and share data. Avoid creating a new catalog for every small project if that makes discovery confusing.

Workspaces attach to a metastore. Unity Catalog workspace bindings can limit which workspaces may use a catalog. This is useful when production data should not be reachable from a development workspace, even if the same metastore serves both.

## Managed or external table?

Use a Unity Catalog managed table when Databricks should manage the table's storage lifecycle and you want platform-managed maintenance. Use an external table when another system owns the files or the storage location must be managed independently. For external data, define storage credentials and external locations rather than putting cloud keys into notebooks.

These terms describe storage ownership. They do not describe whether the data is confidential or whether a table is governed. Both types should have an owner and appropriate permissions.

## Permission model

To read a table, a principal typically needs permission to use its catalog and schema and `SELECT` on the table. Writing needs additional privileges. A simplified example:

```sql
GRANT USE CATALOG ON CATALOG prod_sales TO `sales_analysts`;
GRANT USE SCHEMA ON SCHEMA prod_sales.curated TO `sales_analysts`;
GRANT SELECT ON TABLE prod_sales.curated.orders TO `sales_analysts`;
```

People should receive permissions through identity provider groups. Jobs should run as service principals. Give each group or service principal only the access needed for its work. Check access using the identity that actually runs the job; testing as an administrator can hide missing grants.

Ownership matters too. If every table is owned by an employee who might leave, administration becomes brittle. Assign ownership to an appropriate group or service principal where supported by your operating model.

## Row filters and column masks

A **row filter** decides which rows a user can read. A **column mask** changes or hides the value a user sees in a column. For example, a regional analyst can be limited to their region, while a customer identifier is masked for users who do not need it.

Before adding a policy, decide where the user's region or role comes from and who can change that mapping. Then test with at least two identities: one that should see the row or clear value, and one that should not. Also check writes, views, and downstream jobs so the policy behaves as expected for every consumer.

Views can present a limited set of columns, but they are not a replacement for checking access to the underlying objects. Hiding a column in a dashboard is not an access control.

## Lineage and audit

Lineage helps answer where a table came from and which downstream objects depend on it. Use it before changing a shared table schema. Audit logs and system tables help answer who accessed data, what changed, and which jobs or queries ran. Availability and retention vary by feature and workspace setup, so decide where audit data is stored and who can inspect it.

For sensitive data, record the classification, owner, allowed uses, retention period, and deletion process. A table comment is useful documentation, but it does not enforce privacy rules by itself.

## Secrets and production identities

Do not commit secrets or paste them into notebooks. Use an approved secret manager for any credential that is still required. For automation, prefer OAuth or supported workload identity over a long-lived personal access token. Rotate credentials and keep access to secret scopes or secret managers limited.

## Delta Sharing

Delta Sharing lets a provider grant a recipient access to selected data without first making a separate copy for them. Keep a clear owner for each share. Document the schema and freshness promise, review recipients, and revoke access when it is no longer needed. Plan how you will communicate a breaking schema change.

## A simple access review

For each production table, ask:

1. Who owns the data and approves access?
2. Which groups and jobs can read it?
3. Which identities can modify or drop it?
4. Does it contain personal or confidential data?
5. Do row or column rules apply?
6. Can the access be traced and reviewed later?

## References

- [Unity Catalog](https://docs.databricks.com/aws/en/data-governance/unity-catalog/)
- [Manage privileges](https://docs.databricks.com/aws/en/data-governance/unity-catalog/manage-privileges/)
- [Workspace bindings](https://docs.databricks.com/aws/en/data-governance/unity-catalog/workspace-binding)
- [Row filters and column masks](https://docs.databricks.com/aws/en/data-governance/unity-catalog/filters-and-masks/)
- [External locations](https://docs.databricks.com/aws/en/connect/unity-catalog/cloud-storage/external-locations)
- [Delta Sharing](https://docs.databricks.com/aws/en/delta-sharing/)
