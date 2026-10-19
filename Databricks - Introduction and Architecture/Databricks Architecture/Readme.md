# 📘 Databricks Architecture Terms

## 🔷 1. Control Plane

> Think of this as **Databricks' brain**.

- Managed by Databricks itself (not in your cloud).
- Handles everything like:
  - Web UI (workspace)
  - Job scheduling
  - Authentication and user permissions
  - Cluster configuration interface
- It **does not store or process your data** — it only manages and controls operations.

📌 **Example:**  
You click "Run" in a notebook — Control Plane sends the instruction to start a cluster and execute the code.

---

## 🔷 2. Data Plane

> This is the **working area** where your actual data and code run.

- Lives in your own cloud account (AWS, Azure, or GCP).
- Includes all compute resources (clusters).
- Reads/writes data from your cloud storage (like S3, ADLS, GCS).
- Executes Spark jobs and notebook commands.

📌 **Example:**  
When you run a Spark job in your notebook, it executes in the cluster inside your cloud account (Data Plane).

---

## 🔷 3. Cluster

> A **group of virtual machines** that executes your code.

- Runs in the Data Plane (your cloud).
- You can create:
  - **Interactive Clusters** for development and exploration.
  - **Job Clusters** for scheduled workloads.
- Managed through the Control Plane.

---

## 🔷 4. Notebook

> A web-based coding environment for writing and running code.

- Supports multiple languages like Python, SQL, Scala, etc.
- You can use:
  - Cells for writing code
  - Visualizations for exploring data
  - Widgets for dynamic inputs
- Executes on attached clusters.

---

## 🔷 5. Jobs

> Used to **schedule and automate** your workloads.

- Runs one or more notebooks, scripts, or workflows.
- Supports:
  - Single-task or Multi-task Jobs (workflow pipelines)
  - Retry policies, alerts, notifications
- Scheduled and managed via the Control Plane.

---

## 🔷 6. Repos

> Git integration inside Databricks.

- Connect GitHub, GitLab, or Bitbucket to your workspace.
- Enables source control and team collaboration.
- Use versioned code directly in notebooks.

---

## 🔷 7. DBFS (Databricks File System)

> A built-in file system in Databricks.

- Abstract layer over cloud storage (S3, ADLS, etc.).
- Allows file I/O using familiar file system paths like `/dbfs/...`.
- Supports reading/writing files from notebooks.

📌 **Example:**  
`/dbfs/tmp/my_file.csv`

---

## 🔷 8. DBUtils

> A utility library that helps interact with the Databricks environment.

- Useful for:
  - File operations
  - Secret access
  - Widgets interaction
  - Notebook control

📌 **Example:**  
```python
dbutils.fs.ls("/databricks-datasets")
```

---

## 🔷 9. Magic Commands

> Special commands that simplify notebook operations.

* Prefixed with `%` or `%%`
* Used for file system access, SQL, running other notebooks, etc.

📌 **Examples:**

* `%fs ls /` → List files
* `%sql SELECT * FROM table` → Run SQL
* `%run ./notebook_name` → Run another notebook

---

## 🔷 10. Widgets

> Used to create **interactive notebook inputs**.

* Create dropdowns, text inputs, etc.
* Useful in parameterized notebooks and dashboards.

📌 **Example:**

```python
dbutils.widgets.text("name", "ujjawal")
```

---

## 🔁 Summary Table

| Term           | Simple Meaning                                                 |
| -------------- | -------------------------------------------------------------- |
| Control Plane  | Databricks brain that manages UI, jobs, users, scheduling      |
| Data Plane     | Where your clusters run and process data                       |
| Cluster        | Group of cloud machines to run notebooks and jobs              |
| Notebook       | Coding environment in Databricks (supports multiple languages) |
| Jobs           | Scheduled or manual task runs                                  |
| Repos          | Git-based code repository integration                          |
| DBFS           | Databricks File System (overlay on cloud storage)              |
| DBUtils        | Helper commands for file, secret, and widget operations        |
| Magic Commands | Special notebook shortcuts like `%sql`, `%fs`                  |
| Widgets        | Input controls to make notebooks dynamic                       |

