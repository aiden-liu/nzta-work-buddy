---
name: databricks-tagging
description: >
  Use this skill when the user asks to add, update, remove, or list tags on
  Databricks Unity Catalog objects. Trigger phrases include: "tag a Databricks
  table", "add tags to a catalog", "set tags on a schema", "tag a volume",
  "add column tags", "apply system tags", "remove a tag from", "list tags on",
  "update tags for", "label a Databricks object", "annotate a Unity Catalog
  table". Applies to catalogs, schemas, tables, columns, and volumes.
version: 1.0.0
argument-hint: "[catalog.schema.object] [catalog|schema|table|column|volume] [set|unset|list]"
allowed-tools: [Read, Write, Edit, Bash]
---

# Databricks Tagging Skill

## Overview

This skill generates correct SQL and PySpark code for managing **tags** on
Databricks Unity Catalog objects: **catalogs**, **schemas**, **tables**,
**columns**, and **volumes**.

Tags are key-value string pairs attached to Unity Catalog objects for
governance, cost attribution, data classification, and discoverability.
Always use three-part Unity Catalog naming (`catalog.schema.object`).

---

## When This Skill Applies

- User wants to add, update, or remove tags on any Unity Catalog object
- User asks to list or inspect existing tags on an object
- User is setting up data classification (e.g. PII, sensitivity levels)
- User needs cost attribution or environment labels across objects
- User asks about column-level tagging or system tags

---

## Instructions

### 1. Clarify before generating

If not already clear from context, ask:
- **Object type**: catalog, schema, table, column, or volume
- **Three-part name**: e.g. `main.sales.orders` (column also needs the column name)
- **Operation**: set tags, unset tags, or list tags
- **Tag key-value pairs**: e.g. `('env', 'prod'), ('owner', 'data-eng')`
- **Output format**: SQL DDL or PySpark

---

### 2. Set tags on a catalog

**SQL**
```sql
ALTER CATALOG <catalog_name>
  SET TAGS ('key1' = 'value1', 'key2' = 'value2');
```

**PySpark**
```python
spark.sql("""
  ALTER CATALOG <catalog_name>
  SET TAGS ('key1' = 'value1', 'key2' = 'value2')
""")
```

---

### 3. Set tags on a schema

**SQL**
```sql
ALTER SCHEMA <catalog>.<schema>
  SET TAGS ('key1' = 'value1', 'key2' = 'value2');
```

**PySpark**
```python
spark.sql("""
  ALTER SCHEMA <catalog>.<schema>
  SET TAGS ('key1' = 'value1', 'key2' = 'value2')
""")
```

---

### 4. Set tags on a table

**SQL**
```sql
ALTER TABLE <catalog>.<schema>.<table>
  SET TAGS ('key1' = 'value1', 'key2' = 'value2');
```

**PySpark**
```python
spark.sql("""
  ALTER TABLE <catalog>.<schema>.<table>
  SET TAGS ('key1' = 'value1', 'key2' = 'value2')
""")
```

---

### 5. Set tags on a column

Column tags are set via `ALTER TABLE ... ALTER COLUMN`.

**SQL**
```sql
ALTER TABLE <catalog>.<schema>.<table>
  ALTER COLUMN <column_name>
  SET TAGS ('key1' = 'value1', 'key2' = 'value2');
```

**PySpark**
```python
spark.sql("""
  ALTER TABLE <catalog>.<schema>.<table>
  ALTER COLUMN <column_name>
  SET TAGS ('key1' = 'value1', 'key2' = 'value2')
""")
```

---

### 6. Set tags on a volume

**SQL**
```sql
ALTER VOLUME <catalog>.<schema>.<volume>
  SET TAGS ('key1' = 'value1', 'key2' = 'value2');
```

**PySpark**
```python
spark.sql("""
  ALTER VOLUME <catalog>.<schema>.<volume>
  SET TAGS ('key1' = 'value1', 'key2' = 'value2')
""")
```

---

### 7. Unset (remove) tags

Replace `SET TAGS` with `UNSET TAGS` and provide only the **keys** (no values).

**SQL — table example**
```sql
ALTER TABLE <catalog>.<schema>.<table>
  UNSET TAGS ('key1', 'key2');
```

**SQL — column example**
```sql
ALTER TABLE <catalog>.<schema>.<table>
  ALTER COLUMN <column_name>
  UNSET TAGS ('key1', 'key2');
```

The same `UNSET TAGS` pattern applies to `ALTER CATALOG`, `ALTER SCHEMA`,
and `ALTER VOLUME`.

---

### 8. List / inspect tags

Use `information_schema` views to query tags programmatically.

**Table and column tags**
```sql
-- Tags on tables
SELECT catalog_name, schema_name, table_name, tag_name, tag_value
FROM <catalog>.information_schema.table_tags
WHERE table_name = '<table>';

-- Tags on columns
SELECT catalog_name, schema_name, table_name, column_name, tag_name, tag_value
FROM <catalog>.information_schema.column_tags
WHERE table_name = '<table>';
```

**Catalog and schema tags**
```sql
-- Tags on schemas
SELECT catalog_name, schema_name, tag_name, tag_value
FROM <catalog>.information_schema.schema_tags
WHERE schema_name = '<schema>';

-- Tags on catalogs (system catalog)
SELECT catalog_name, tag_name, tag_value
FROM system.information_schema.catalog_tags
WHERE catalog_name = '<catalog>';
```

**Volume tags**
```sql
SELECT catalog_name, schema_name, volume_name, tag_name, tag_value
FROM <catalog>.information_schema.volume_tags
WHERE volume_name = '<volume>';
```

**PySpark — generic helper**
```python
tags_df = spark.sql("""
  SELECT catalog_name, schema_name, table_name, tag_name, tag_value
  FROM <catalog>.information_schema.table_tags
  WHERE table_name = '<table>'
""")
tags_df.show(truncate=False)
```

---

### 9. Bulk-tag multiple objects (PySpark loop)

When the user needs to apply the same tags to many objects at once:

```python
tables = [
    "main.sales.orders",
    "main.sales.customers",
    "main.sales.products",
]

tags = ("('env', 'prod'), ('owner', 'data-eng'), ('classification', 'internal')")

for table in tables:
    spark.sql(f"ALTER TABLE {table} SET TAGS {tags}")
    print(f"Tagged {table}")
```

---

## Best Practices to Follow

- Tag keys and values are **case-sensitive strings** — agree on a naming
  convention (e.g. `snake_case`) before applying tags at scale
- Use tags for **data classification** (`pii = true`, `sensitivity = high`),
  **cost attribution** (`team = analytics`, `project = q3-launch`), and
  **lifecycle** (`env = prod`, `status = deprecated`)
- Tags on a parent object (catalog, schema) are **not** automatically inherited
  by child objects — apply tags explicitly at each level if needed
- Use `information_schema` views rather than `DESCRIBE` for programmatic
  tag auditing across many objects
- Remind the user that tagging requires `APPLY TAG` privilege on the object
  in Unity Catalog (granted separately from `SELECT` / `MODIFY`)
- When unsetting tags, only keys are required — values are ignored by `UNSET TAGS`

---

## Permissions Required

| Operation | Required Privilege |
|---|---|
| Set / unset tags on a table or column | `APPLY TAG` on the table |
| Set / unset tags on a schema | `APPLY TAG` on the schema |
| Set / unset tags on a catalog | `APPLY TAG` on the catalog |
| Set / unset tags on a volume | `APPLY TAG` on the volume |
| Read tags via `information_schema` | `SELECT` on the `information_schema` view |

Offer to generate the corresponding `GRANT APPLY TAG` statement when the user
is setting up tagging for a team.

**SQL — grant example**
```sql
GRANT APPLY TAG ON TABLE <catalog>.<schema>.<table> TO <principal>;
```

---

## Common Follow-ups to Offer

After generating the requested tagging code, offer to also generate:
- `GRANT APPLY TAG` statements so team members can manage tags
- A tag audit query across an entire schema or catalog using `information_schema`
- A bulk-tagging PySpark loop if the user needs to tag many objects
- Terraform equivalents (`databricks_catalog_tag`, `databricks_schema_tag`,
  `databricks_table_tag`) if the user manages infra-as-code
- A `DESCRIBE EXTENDED` snippet to verify tags were applied correctly
