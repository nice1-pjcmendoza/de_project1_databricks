# 🗂️ Build the Catalog & Schemas

We begin the project by creating a Catalog and Schemas in Databricks. The Catalog is a logical container for databases, and the Schemas are logical containers for tables. This structure allows us to organize our data according to the Medallion Architecture, which consists of the Landing, Bronze, Silver, and Gold layers.

## Create the *cyclistic* Catalog

We create a Catalog `cyclistic` that will serve as the top-level container for all our data. The `MANAGED LOCATION` points to the root of our storage account (i.e. `deprojectcontainer`), and we will create an external volume in the Landing layer to reference specific folders in our storage account.

```sql
CREATE CATALOG IF NOT EXISTS cyclistic 
MANAGED LOCATION 'abfss://deprojectcontainer@deprojectextdatalake.dfs.core.windows.net/'
COMMENT 'Catalog for the Cyclistic project';
```

Let's verify the catalog we just created.

```sql
DESCRIBE CATALOG EXTENDED cyclistic;
```

<details>
<summary>👉 View images here</summary>

Here's the output from Databricks:

![1771130448904](image/README/1771130448904.png)

You can also view its properties in the Databricks UI:

![1771130523306](image/README/1771130523306.png)

</details>


## Create the Schemas

Following the Medallion Architecture, we will create four schemas: `landing`, `bronze`, `silver`, and `gold`.

If you want to learn more about the Medallion Architecture, check out this [blog post](https://www.datawithbaraa.com/post/medallion-architecture-on-databricks).

```sql
USE CATALOG cyclistic;

CREATE SCHEMA IF NOT EXISTS landing COMMENT 'Raw CSV files';
CREATE SCHEMA IF NOT EXISTS bronze COMMENT 'Raw ingested data';
CREATE SCHEMA IF NOT EXISTS silver COMMENT 'Cleansed, conformed data';
CREATE SCHEMA IF NOT EXISTS gold   COMMENT 'For analytics (facts, dimensions, aggregations)';
```