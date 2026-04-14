---
name: databricks-provisioning
description: >
  Use this skill when the user asks to create, define, or scaffold Databricks
  objects such as tables, volumes, schemas, or catalogs. Trigger phrases include:
  "create a Databricks table", "add a volume", "set up a Unity Catalog schema",
  "create a catalog", "define a Delta table", "provision a managed/external table",
  "write the DDL for", "create a volume in Databricks", "scaffold a Databricks table".
  Applies to both SQL DDL and PySpark DataFrame / DeltaTable API approaches.
version: 1.0.0
argument-hint: "[catalog.schema.object] [managed|external] [sql|pyspark]"
allowed-tools: [Read, Write, Edit, Bash]
---

# Databricks Provisioning Skill

## Overview

This skill generates correct, production-ready SQL DDL and PySpark code for
provisioning Databricks Unity Catalog objects: **catalogs**, **schemas**,
**tables** (managed and external), and **volumes**.

Always prefer Unity Catalog three-part naming (`catalog.schema.object`).
Default to Delta format unless the user specifies otherwise.

---

## When This Skill Applies

- User wants to create or define a Databricks table, volume, schema, or catalog
- User asks for DDL, CREATE statements, or PySpark provisioning code
- User is scaffolding a new data pipeline and needs the storage layer defined
- User asks about managed vs. external tables or volume types

---

## Instructions

### 1. Clarify before generating

If not already clear from context, ask:
- **Three-part name**: catalog, schema, object name (e.g. `main.sales.orders`)
- **Table type**: managed (default) or external (requires `LOCATION`)
- **Volume type**: managed (default) or external (requires `LOCATION`)
- **Output format**: SQL DDL, PySpark, or both
- **Columns / schema**: if creating a table, ask for columns or infer from context

### 2. Catalog

**SQL**
```sql
CREATE CATALOG IF NOT EXISTS <catalog_name>
  COMMENT '<optional description>';
```

**PySpark**
```python
spark.sql("CREATE CATALOG IF NOT EXISTS <catalog_name> COMMENT '<desc>'")
```

### 3. Schema

**SQL**
```sql
CREATE SCHEMA IF NOT EXISTS <catalog>.<schema>
  COMMENT '<optional description>';
```

**PySpark**
```python
spark.sql("CREATE SCHEMA IF NOT EXISTS <catalog>.<schema> COMMENT '<desc>'")
```

### 4. Managed Table

Databricks manages the data lifecycle; no `LOCATION` clause.

**SQL**
```sql
CREATE TABLE IF NOT EXISTS <catalog>.<schema>.<table> (
  id        BIGINT NOT NULL,
  name      STRING,
  created_at TIMESTAMP DEFAULT current_timestamp()
)
USING DELTA
COMMENT '<optional description>'
TBLPROPERTIES (
  'delta.autoOptimize.optimizeWrite' = 'true',
  'delta.autoOptimize.autoCompact'   = 'true'
);
```

**PySpark**
```python
from delta.tables import DeltaTable

DeltaTable.createIfNotExists(spark) \
  .tableName("<catalog>.<schema>.<table>") \
  .addColumn("id",         "BIGINT", nullable=False) \
  .addColumn("name",       "STRING") \
  .addColumn("created_at", "TIMESTAMP") \
  .property("delta.autoOptimize.optimizeWrite", "true") \
  .property("delta.autoOptimize.autoCompact",   "true") \
  .comment("<optional description>") \
  .execute()
```

### 5. External Table

User must supply an external storage path (ADLS, S3, GCS).

**SQL**
```sql
CREATE TABLE IF NOT EXISTS <catalog>.<schema>.<table> (
  id   BIGINT NOT NULL,
  name STRING
)
USING DELTA
LOCATION '<abfss://container@account.dfs.core.windows.net/path>'
COMMENT '<optional description>';
```

**PySpark**
```python
DeltaTable.createIfNotExists(spark) \
  .tableName("<catalog>.<schema>.<table>") \
  .location("<abfss://container@account.dfs.core.windows.net/path>") \
  .addColumn("id",   "BIGINT", nullable=False) \
  .addColumn("name", "STRING") \
  .comment("<optional description>") \
  .execute()
```

### 6. Managed Volume

Stores arbitrary files (CSV, JSON, images, etc.) under Unity Catalog governance.

**SQL**
```sql
CREATE VOLUME IF NOT EXISTS <catalog>.<schema>.<volume>
  COMMENT '<optional description>';
```

**PySpark**
```python
spark.sql("""
  CREATE VOLUME IF NOT EXISTS <catalog>.<schema>.<volume>
  COMMENT '<optional description>'
""")
```

Access path: `/Volumes/<catalog>/<schema>/<volume>/`

### 7. External Volume

Points to an existing cloud storage path via a Storage Credential + External Location.

**SQL**
```sql
CREATE EXTERNAL VOLUME IF NOT EXISTS <catalog>.<schema>.<volume>
  URL '<abfss://container@account.dfs.core.windows.net/path>'
  COMMENT '<optional description>';
```

**PySpark**
```python
spark.sql("""
  CREATE EXTERNAL VOLUME IF NOT EXISTS <catalog>.<schema>.<volume>
  URL '<abfss://container@account.dfs.core.windows.net/path>'
  COMMENT '<optional description>'
""")
```

---

## Best Practices to Follow

- Always use `IF NOT EXISTS` to make statements idempotent
- Always use three-part Unity Catalog naming (`catalog.schema.object`)
- Default to **Delta** format for tables
- Add `COMMENT` fields for discoverability
- For managed tables, include `autoOptimize` table properties
- For external objects, remind the user that a **Storage Credential** and
  **External Location** must be configured in Unity Catalog first
- When generating PySpark, import `from delta.tables import DeltaTable` for
  table creation; use `spark.sql(...)` for catalog/schema/volume DDL
- Suggest `GRANT` statements when the user is setting up objects for a team

---

## Common Follow-ups to Offer

After generating the requested object, offer to also generate:
- `GRANT` / `REVOKE` permission statements
- A `DESCRIBE DETAIL` or `DESCRIBE EXTENDED` snippet to verify the object
- A sample `INSERT` or file-upload snippet to validate the table/volume works
- Terraform equivalents (`databricks_schema`, `databricks_table`, `databricks_volume`) if the user manages infra-as-code
