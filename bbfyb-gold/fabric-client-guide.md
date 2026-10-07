<!-- Published copy, generated from WeedCrawler's private repository. Do not edit here: changes are overwritten on the next publish. -->
# Connecting Microsoft Fabric to your BBFYB data feed

This guide walks you through connecting your Microsoft Fabric workspace to the BBFYB
(Best Bang For Your Bud) retail cannabis dataset. The data is delivered as **Apache
Iceberg tables** hosted in cloud storage: you create a *shortcut* to it from Fabric —
no copies, no pipelines, no database credentials. Once connected, the tables behave
like native Fabric tables in Power BI, SQL, and Spark, and they update automatically
as BBFYB refreshes the data.

**Time required:** about 15 minutes.

## What you'll receive from BBFYB

Before starting, you should have received (via a secure channel):

1. **Storage endpoint URL** — looks like
   `https://bbfyb-gold-iceberg-snowflake.s3.ca-central-1.amazonaws.com`
2. **Access key ID** and **secret access key** — read-only credentials for the feed
3. **A list of table folders** you're subscribed to. Folder names include a
   system-generated suffix — use them exactly as provided, e.g.:
   - `gold/dim_master_store.WNFthPzE/` — one row per retail store (name, banner, city, province, geo)
   - `gold/fct_sales_daily_sample.Xhuq1c1t/` — daily sales by store × product variant

If any of these are missing, contact BBFYB before proceeding.

## Prerequisites on your side

- A Microsoft Fabric **capacity** (any SKU, including trial) and a workspace where
  you can create items.
- Permission to create **connections** in Fabric (Admin portal → most tenants allow
  this by default).

## Step 1 — Create a Lakehouse

1. Open your Fabric workspace → **+ New item** → **Lakehouse**.
2. Name it (e.g. `bbfyb_data`).
3. **Important:** leave the **"Lakehouse schemas"** checkbox **unchecked**. The
   automatic Iceberg-to-Delta conversion requires a lakehouse *without* schemas
   enabled. If you already have a schema-enabled lakehouse, create a new one for
   this feed.

## Step 2 — Create the connection

1. In the Lakehouse, find the **Tables** section in the left explorer.
2. Click the **…** menu next to Tables → **New shortcut**.
3. Choose **Amazon S3** as the source.
4. Select **Create new connection** and fill in:
   - **URL:** the storage endpoint URL from BBFYB
   - **Connection name:** `BBFYB feed` (or anything you like)
   - **Authentication kind:** Access Key
   - **Access Key ID / Secret Access Key:** the credentials from BBFYB
5. Click **Next**.

## Step 3 — Create one shortcut per table

1. In the folder browser, navigate into `gold/`.
2. Tick the checkbox next to a **table folder** (e.g. `dim_master_store.WNFthPzE`) —
   the folder that directly contains `data` and `metadata` subfolders. Do **not**
   select the `data` or `metadata` subfolders themselves, and do **not** select
   the parent `gold` folder.
3. Click **Create**. Repeat for each table folder you're subscribed to.

## Step 4 — Verify

Within a minute or two, each shortcut appears under **Tables** as a regular
(Delta) table.

- Click a table to preview rows.
- To confirm the conversion succeeded: right-click the table → **View files** →
  you should see a `_delta_log/` folder containing `latest_conversion_log.txt`;
  open it to see the conversion status.

If a table shows as a plain folder instead of a table, see Troubleshooting below.

## Step 5 — Use the data

- **Power BI:** the Lakehouse's **SQL analytics endpoint** exposes the tables to
  Power BI directly — build your semantic model on top of it like any other source.
- **SQL:** switch to the SQL analytics endpoint (top-right of the Lakehouse view)
  and query, e.g.:

  ```sql
  SELECT d.PROVINCE_STATE, SUM(f.SALES_DOLLARS) AS sales
  FROM fct_sales_daily_sample f
  JOIN dim_master_store d ON d.MASTER_STORE_ID = f.MASTER_STORE_ID
  GROUP BY d.PROVINCE_STATE
  ORDER BY sales DESC;
  ```
- **Spark notebooks:** the tables are readable as Delta. If you ever see a decimal
  conversion error in Spark, set
  `spark.conf.set("spark.sql.parquet.enableVectorizedReader", "false")` and re-run.

## What to expect from the feed

- **Refresh:** BBFYB refreshes daily in the morning (Eastern Time). New data is
  visible in Fabric within minutes of the refresh — no action needed on your side.
- **Revisions window:** figures for the most recent **7 days** are provisional and
  may be revised as late-arriving data lands; day 8 onward is stable.
- **`IS_SYNTHETIC` column (sales table):** rows flagged `TRUE` are modeled estimates
  BBFYB fills in when a store's feed was temporarily unavailable, so totals stay
  representative. Filter them out with `WHERE IS_SYNTHETIC = FALSE` if you only want
  directly observed transactions.
- **Units are fractional** by design (estimation model). Round *after* aggregating,
  not per row.

## Troubleshooting

| Symptom | Fix |
|---|---|
| Shortcut shows as a folder, not a table | The shortcut probably targets the wrong level. Delete it and re-create it pointing at the table folder itself (the one containing `data` + `metadata`). |
| No `_delta_log` / no conversion log | Conversion wasn't attempted — most often the lakehouse is schema-enabled, or the shortcut isn't directly under **Tables**. Re-create in a non-schema lakehouse, directly under Tables. |
| "Access denied" when browsing folders | Credentials typo, or your Fabric tenant blocks outbound connections — check with your Fabric admin, then verify the access key/secret. |
| Decimal/parquet error in Spark | Disable the vectorized reader (see Step 5). |
| Table stopped updating | Check `latest_conversion_log.txt` (View files) for the last conversion; if it shows an error, contact BBFYB. |

## Questions?

Contact BBFYB at **hi@weedcrawler.ca** with the table name and, if relevant, the
contents of `latest_conversion_log.txt`.
