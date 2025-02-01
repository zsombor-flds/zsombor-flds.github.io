---
date: 2025-02-01T14:09:03+01:00
description: ""
lastmod: 2025-02-01
showTableOfContents: false
tags:
  - Databricks
  - Polars
  - Delta Lake
  - Unity Catalog
title: How to handle Databricks Deletion Vectors in Python with Polars
type: post
---
## Accessing Unity Catalog with Polars

Last week, I came across a great [post](https://substack.com/@dataengineeringcentral/p-155796796) from [Daniel Beach](https://www.linkedin.com/in/daniel-beach-6ab8b4132/) on how to access objects in Unity Catalog from DuckDB, Polars. However, when I wanted to replicate the Polars setup I hit unexpected limitations in how Databricks handles Delta Lake feature compatibility under the hood.
I was surprised to see features that are currently only supported by Databricks, which limit users from accessing Delta Lake tables with different clients.

I wanted to document my way of finding a workaround for these limitations.
My first goal was to just execute simple scripts that read from a delta table managed by Unity Catalog. Because Databricks open-sourced Unity Catalog it shouldn't matter but in my case, I access the Unity Catalog in a sandbox DBX environment hosted In Azure. 

The demo script looks like this:

```python
import json
import polars as pl

BEARER_TOKEN = "xxxxxx"
DBX_HOST = "https://xxxxxx.azuredatabricks.net/"
UC_NAME = "sandbox_uc"

def demo(schema, table) -> pl.DataFrame:
    """
    This function attaches to the UC catalog using the provided Databricks host and token,
    prints the table information, scans a table, and executes an SQL query on it.
    Returns:
        pl.DataFrame: The processed DataFrame containing the queried table data.
    """
    # Attach to the UC catalog
    catalog = pl.Catalog(DBX_HOST, bearer_token=BEARER_TOKEN)

    # Retrieve and log single table information
    table_info = catalog.get_table_info(UC_NAME, schema, table)
    print("Single table info:")
    print(json.dumps(table_info, indent=4))

    # Scan the table and process data
    df = catalog.scan_table(UC_NAME, schema, table)
    df = (
        df.sql("SELECT * FROM self")
          .collect()
    )

    return df

if __name__ == "__main__":
    schema = 'fk'
    table = 'customers'
    df = demo(schema, table)
    print(df)
```

 When I first executed the script, I received the following error:

```python
    return scan_delta(
           ^^^^^^^^^^^
  File ".venv/lib/python3.12/site-packages/polars/io/delta.py", line 366, in scan_delta
    raise DeltaProtocolError(msg)
_internal.DeltaProtocolError: The table has set these reader features: {'deletionVectors'} but these are not yet supported by the polars delta scanner.
```
So what's happened here? 
I was able to connect to the specified Unity Catalog with Polars, and I was able to fetch metadata information with `catalog.get_table_info`, but I was unable to read the dam file. Handling [deletionVectors](https://docs.databricks.com/en/delta/deletion-vectors.html) is not (yet) supported by the [delta rs](https://github.com/delta-io/delta-rs) library therefore Polars is unable to read from this object

## What the heck are deletion vectors? 
Deletion vectors are actually fancy mechanisms for marking rows as deleted without rewriting the underlying data files.  By storing these deletions as metadata separate from the actual data, deletion vectors are generally more cost-effective compared to traditional file rewrites.
This is important for Delta Lake because it streamlines the process of handling data updates and deletions without the heavy overhead of rewriting entire files.  However unfortunately as of January 31, 2024, deletion vectors are not yet supported in Polars.
## Resolution

As you can see in the following screenshot, the delta table that I created with Databricks SQL has the `delta.feature.deletionVectors` property set to *enabled*. 

![](/images/dbx-deletionVectors.png)

Since the deletionVectors feature appeared in the table properties, my first instinct was to modify or unset it.
However, both of the following queries were unsuccessful

```sql
alter table customers set tblproperties ('delta.feature.enabledeletionvectors' = false);
-- or
alter table customers unset tblproperties ('delta.feature.enabledeletionvectors');
```
Since deletion vectors are enforced at the feature level (rather than just a table property), we need to simply unset it.

What worked?  I found out that Databricks has a fairly detailed write-up on how they handle compatibility with the OSS Delta Lake protocol. You can read more about it here: [# How does Databricks manage Delta Lake feature compatibility](https://docs.databricks.com/en/delta/feature-compatibility.html#how-does-databricks-manage-delta-lake-feature-compatibility)

Based on this, I realized that you can [drop](https://docs.databricks.com/en/delta/drop-feature.html )/disable features to ensure compatibility with Delta Lake clients.

```sql
alter table customers
drop feature deletionVectors
truncate history;
```

I am not going to lie; this query is definitely on the podium in the contest of furthest from ANSI SQL statements -- but at least it's an option. One important pitfall is that you have to wait 24 hours for the retention period to expire (24 hours) and by **truncating the history you will lose the ability to use time travel** to access older snapshots of your data, as all transaction logs are truncated.

My takeaway for now is that if you're planning to work with Delta tables outside of Databricks, to have fewer surprises, you need to keep an eye on feature compatibility to ensure smoother integrations, but hopefully, smoother integrations will come in the future.