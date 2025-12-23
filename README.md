# bankrisk01 — Lightweight Azure Bank Risk Data Engineering Project (Production-Style, Low Cost)

## What this is
**bankrisk01** is a simple end-to-end Azure data engineering pipeline that ingests a public finance dataset (Treasury yield curve rates), lands it in **ADLS Gen2**, transforms it with **Azure Databricks**, and queries it with **Azure Synapse Analytics (Serverless SQL)**.

It’s intentionally lightweight:
- No SQL databases
- No complicated business logic
- Still “real” production-style building blocks: RBAC, managed identities, Key Vault, orchestration, medallion-ish layout, and serverless querying

## Why this helps you get a bank data engineering job
Banks care about:
- Secure ingestion (no hardcoded secrets)
- Repeatable pipelines + orchestration
- Data lake organization + standard formats (Parquet/Delta)
- Transformations that create analytics-ready datasets
- Query layers for analysts / BI / downstream consumers

This project demonstrates those fundamentals without drowning you in complexity.

---

# Architecture (simple, real, production-style)

**Source (Public API)**  
US Treasury Fiscal Data API (Yield Curve Rates)

**Ingestion (ADF)**  
Copy API response → ADLS `raw/`

**Transform (Databricks)**  
Raw JSON/CSV → cleaned Delta/Parquet in `curated/` and `analytics/`

**Query (Synapse Serverless SQL)**  
Query files in ADLS directly with `OPENROWSET`

---

## High-level flow
1. **ADF** calls API (parameterized date range)
2. **ADF** writes file into **ADLS Gen2**: `raw/treasury_yield_curve/dt=YYYY-MM-DD/`
3. **ADF** triggers **Databricks Notebook**
4. **Databricks** reads raw files, cleans/casts, creates a few simple “risk indicators”
5. **Databricks** writes curated + analytics outputs
6. **Synapse Serverless** queries the analytics output (no database costs)

---

# Services you will use
- **Azure Data Lake Storage Gen2 (ADLS)**: cheap storage + folder structure
- **Azure Key Vault**: store secrets (service principal) and keep notebooks clean
- **Azure Data Factory (ADF)**: orchestration + ingestion + notebook triggering
- **Azure Databricks (ADB)**: transformation, Delta/Parquet outputs
- **Azure Synapse Analytics (Serverless SQL)**: query over files (pay-per-query)
---

# Cost control (do this or you’ll waste money)
- Databricks cluster: **single node**, **smallest VM available**, **auto-terminate 10–15 minutes**
- Synapse: **Serverless SQL only** (avoid dedicated pools)
- ADF triggers: run daily or manual while learning
- Storage: keep only a small time window in raw (e.g., last 30–90 days)
---

# Dataset (simple “bank risk” framing)
We use **Treasury yield curve rates** as a basic market-risk indicator:
- Build **2y–10y spread** (inversion check)
- Track daily changes and rolling stats (optional)

This is “bank risk” without complicated models.
---

# Naming (keep it consistent)
Use these names (adjust to make globally unique where required):

- Resource Group: `rg-bankrisk01`
- Storage account: `stbankrisk01<unique>`  (must be globally unique)
- Key Vault: `kv-bankrisk01<unique>`
- Data Factory: `adf-bankrisk01`
- Databricks workspace: `adb-bankrisk01`
- Synapse workspace: `syn-bankrisk01<unique>`
---


# Step 0 — Prereqs
You need:
- An Azure subscription where you can create resources
- Permission to assign RBAC roles
- A Microsoft Entra ID tenant (normal Azure login is fine)

Recommended local tools (optional):
- Azure CLI
- VS Code
---

<br>


# Step 1 — Create Resource Group
Azure Portal:
1. Search **Resource groups**
2. **Create**
3. Name: `rg-bankrisk01`
4. Region: pick one and stick with it (same region for everything)
5. Create
---

<br>


# Step 2 — Create ADLS Gen2 (Storage Account with HNS)
Azure Portal:
1. Search **Storage accounts** → **Create**
2. Subscription: your subscription
3. Resource group: `rg-bankrisk01`
4. Storage account name: `stbankrisk01<unique>`
5. Region: same as RG
6. Performance: **Standard**
7. Redundancy: **LRS** (cheapest)
8. Go to **Advanced** tab:
   - Enable **Hierarchical namespace** = **ON** (this makes it ADLS Gen2)
9. Create

## Create containers (keep it very simple)
Go to the storage account → **Data storage** → **Containers** → create:

- `raw`
- `curated`
- `analytics`
- `logs` (optional)

## Create folders (simple)
You don’t need to pre-create folders, but here is the target layout:

- `raw/treasury_yield_curve/dt=YYYY-MM-DD/`
- `curated/treasury_yield_curve/`
- `analytics/bankrisk_indicators/`
---

<br>


# Step 3 — Create Key Vault (and secrets)
Azure Portal:
1. Search **Key vaults** → **Create**
2. Resource group: `rg-bankrisk01`
3. Key vault name: `kv-bankrisk01<unique>`
4. Region: same as everything else
5. Pricing tier: **Standard**
6. Create

## Key Vault settings you should turn on (basic hardening)
Inside the Key Vault:
- **Soft-delete**: ON (usually default)
- **Purge protection**: ON if allowed in your subscription (good practice)

## Add secrets (we’ll use a service principal for Databricks -> ADLS access)
We will create a service principal (app registration) and store:
- `sp-client-id`
- `sp-client-secret`
- `tenant-id`
- `storage-account-name`

### Create Service principal (Microsoft Entra ID)
Azure Portal:
1. Search **Microsoft Entra ID**
2. Go to **App registrations** → **New registration**
3. Name: `sp-bankrisk01`
4. Register

### Create New Client Secret
1. Still under Microsoft Entra ID, Go to the app → **Overview**
   - Copy **Application (client) ID** → store it for Key Vault
   - Copy **Directory (tenant) ID** → store it for Key Vault
2. Go to Manage → **Certificates & secrets**
   - **New client secret**
   - Copy the *value* immediately (you won’t see it again)

### Add Role assignment
 - Go to Key Vault 'kv-bankrisk01' → Access Control (IAM) → Add → Add role assignmen
 - Role: **Key Vault Administrator**
 - Members: User, group, or service principal
 - Click 'Review and Assign'
   
### Put those values into Key Vault secrets
Key Vault 'kv-bankrisk01' → **Objects**  → **Secrets** → **Generate/Import**

Create:
- Name: `sp-client-id` → Value: (client id)
- Name: `sp-client-secret` → Value: (client secret value)
- Name: `tenant-id` → Value: (tenant id)  ---- DON'T SEE THIS OPTION
- Name: `storage-account-name` → Value: `stbankrisk01<unique>`  ---- DON'T SEE THIS OPTION
---

<br>


# Step 4 — RBAC permissions (don’t skip this)

## 4A) Enable ADF Managed Identity
ADF → **Manage** (toolbox icon) → **Managed identities**
- System assigned: **ON**
- Save

We need secure access between services:

## 4B) Grant the service principal access to ADLS
Storage account → **Access Control (IAM)** → **Add role assignment**
- Role: **Storage Blob Data Contributor**
- Members: User, group, or service principal
- Select: `sp-bankrisk01`
- Save

This lets Databricks write to ADLS using the service principal creds stored in Key Vault.
---
<br>

# Step 5 — Create Azure Data Factory (ADF)
Azure Portal:
1. Search **Data factories** → **Create**
2. Resource group: `rg-bankrisk01`
3. Name: `adf-bankrisk01`
4. Region: same
5. Create Create

  
## Step 5a) Give ADF permission to **WRITE** to ADLS Gen2 (**Storage Blob Data Contributor**)
   - Go to your Storage account (the ADLS Gen2 one)
   - Left menu → Access Control (IAM)
   - Click Add → Add role assignment
   - Role tab: **Storage Blob Data Contributor**
   - Members tab:
     - “Assign access to” = Managed identity
     - Select members
        - Managed identity type = Data Factory
        - Pick your factory: adf-bankrisk01
   - Review + assign
   - What you should see after:
     - In Storage account → IAM → Role assignments:
       - adf-bankrisk01 (Managed Identity) → Storage Blob Data Contributor
       
## Step 5b) Give ADF permission to **READ** secrets from Key Vault (**Key Vault Secrets User**)
   - Go to your Storage account (the ADLS Gen2 one)
   - Left menu → Access Control (IAM)
   - Click Add → Add role assignment
   - Role tab: **Key Vault Secrets User**
   - Members tab:
     - “Assign access to” = Managed identity
     - Select members
        - Managed identity type = Data Factory
        - Pick your factory: adf-bankrisk01
   - Review + assign
   - What you should see after:
     - In Storage account → IAM → Role assignments:
       - adf-bankrisk01 (Managed Identity) → Key Vault Secrets User
---
<br>

# Step 6 — ADF Linked Services
Open ADF Studio (Launch Studio).

## 6A) Linked Service: ADLS Gen2
1. Manage → **Linked services** → **New**
2. Choose **Azure Data Lake Storage Gen2**
3. Auth method: **Managed Identity**
4. Select your storage account
5. Test connection → Create

Name it: `LS_ADLS_bankrisk01`

## 6B) Linked Service: HTTP (Treasury Fiscal Data API)
1. Linked services → New
2. Choose **HTTP**
3. Name: `LS_HTTP_TreasuryFiscalData`
4. Base URL: `https://api.fiscaldata.treasury.gov/services/api/fiscal_service/`
5. Authentication type: `Anonymous authentication`
6. Test connection → Create


## 6C) Linked Service: Azure Databricks
1. Linked services → New
2. Choose **Azure Databricks DeltaLake**
3. Workspace: your `adb-bankrisk01`
4. Authentication:
   - Use a personal access token and store it in Key Vault (best practice)
    Step 1) In Azure Databricks:
     - Open your Databricks workspace
     - Click your user icon/name (top bar) → Settings
     - Developer
     - Next to Access tokens → Manage
     - Generate new token
     - Add a comment like: bankrisk01-adf-linkedservice
     - Set lifetime (pick something sane; banks rotate these)
     - Copy the token value immediately (you won’t see it again)
   Step 2) Store the PAT in Azure Key Vault as a secret
     - In Key Vault:
     - Secrets → Generate/Import
     - Name: adb-pat
     - Value: paste the PAT
     - Create
     - If you get “operation not allowed by RBAC”, you need a Key Vault data-plane role
        like Key Vault Secrets Officer (to create/update secrets). “Key Vault Contributor” often cannot create secrets.
    Step 3 — Give ADF Managed Identity permission to read that secret
      3A) Turn on ADF System Assigned Managed Identity
      ADF → Manage → Managed identities:
       - System assigned: On → Sav
      3B) Assign Key Vault read permissions to ADF identity (RBAC model)
       - Key Vault → Access Control (IAM) → Add role assignment:
       - Role: Key Vault Secrets User
       - Members: select your Data Factory managed identity (adf-bankrisk01)
       - This role lets ADF Get/List secrets (read-only), which is exactly what you want.
   Step 4 — Create a Key Vault Linked Service in ADF
   In ADF Studio:
     - Manage → Linked services → New
     - Choose Azure Key Vault
     - Authentication method: Managed identity
     - Select your vault
     - test connection → Create
   Name it something like: LS_KV_bankrisk01
   Step 5 — Create the Azure Databricks Linked Service in ADF using the Key Vault secret
   In ADF Studio:
     - Manage → Linked services → New
     - Choose Azure Databricks
     - Fill in the normal fields:
       -   Databricks workspace URL (looks like https://adb-<id>.<region>.azuredatabricks.net)
       -   Choose cluster / interactive / job cluster config as you prefer
     - Authentication type: Access token
     - For the Access token value:
       - choose Azure Key Vault (or “Use secret” / “Key Vault reference” depending on UI)
       - select Key Vault linked service: LS_KV_bankrisk01
       - Select secret name: adb-pat


---
<br>

# Step 7 — ADF Pipeline (Ingest + Trigger Databricks)
Create pipeline: `PL_bankrisk01_ingest_and_transform`

## 7A) Pipeline parameters
Add parameters:
- `pRunDate` (String) default: `@{formatDateTime(utcnow(),'yyyy-MM-dd')}`
- `pDaysBack` (Int) default: `30`

## 7B) Activity 1 — Copy data from API to ADLS raw
Add **Copy data** activity: `CP_TreasuryYieldCurve_ToRaw`

### Source (HTTP dataset)
Create a dataset for HTTP:
- Linked service: `LS_HTTP_TreasuryFiscalData`
- Relative URL example (we’ll parameterize it):

**Important note:** Treasury API endpoints evolve. We’ll build this so you only change one string if needed.

Use a relative URL like (example pattern):
`v2/accounting/od/avg_interest_rates`

If you prefer yield curve specifically, swap to the dataset your API returns for yield curve rates.
The pattern is identical: ADF calls an endpoint, lands the response in ADLS.

**Query parameters** (conceptually):
- filter for last N days
- choose CSV or JSON

For simplicity, keep JSON.

### Sink (ADLS Gen2 dataset)
Create ADLS dataset:
- Linked service: `LS_ADLS_bankrisk01`
- File path (dynamic):
  - Container: `raw`
  - Directory: `treasury_yield_curve/dt=@{pipeline().parameters.pRunDate}`
  - File: `treasury_yield_curve_@{pipeline().parameters.pRunDate}.json`

If ADF UI doesn’t allow full expression directly in the dataset path, set the dataset to a base folder and use dynamic content in the Copy activity sink path.


## 7C) Activity 2 — Databricks Notebook
Add **Databricks Notebook** activity: `NB_Transform_TreasuryYieldCurve`

- Linked service: `LS_ADB_bankrisk01`
- Notebook path: (you’ll create it in Databricks, see next section)
- Base parameters:
  - `run_date` = `@{pipeline().parameters.pRunDate}`

## 7D) Trigger
While learning: start with **Manual trigger**.
Later: add a daily schedule trigger.
---
<br>

# Step 8 — Create Azure Databricks
Azure Portal:
1. Search **Azure Databricks** → **Create**
2. Resource group: `rg-bankrisk01`
3. Name: `adb-bankrisk01`
4. Pricing tier: choose the lowest you can (many subscriptions have a default)
5. Create

## Create a small cluster (keep it cheap)
Databricks workspace → **Compute** → **Create compute**
- Single node: **ON**
- Node type: smallest available
- Auto-terminate: **10–15 minutes**
- Runtime: latest LTS is fine
---

<br>


# Step 9 — Databricks Secrets (Key Vault-backed)
We want zero secrets in notebooks.

## 9A) Create a Key Vault-backed secret scope
Databricks workspace:
1. Go to **Secrets** (or use Databricks CLI/UI depending on workspace)
2. Create a secret scope backed by Azure Key Vault:
   - Scope name: `kv-bankrisk01-scope`
   - DNS name: your Key Vault URI
   - Resource ID: Key Vault resource ID

Once created, Databricks can read Key Vault secrets via:
`dbutils.secrets.get(scope="kv-bankrisk01-scope", key="sp-client-id")`
---

<br>


# Step 10 — Databricks Notebook Code (Python)
Create a notebook in Databricks:
- Name: `bankrisk01_transform`
- Language: Python
- Put the code below into it.

This notebook:
- Reads raw JSON from ADLS
- Extracts rates fields (example schema)
- Creates a simple risk indicator: `spread_2y_10y` and an inversion flag
- Writes curated + analytics outputs back to ADLS

> You may need to adjust the input schema depending on the exact API dataset you chose.
> The structure is the point: read raw → normalize → write curated/analytics.

```python
from pyspark.sql import functions as F
from pyspark.sql import types as T

# -----------------------------
# PARAMETERS (from ADF)
# -----------------------------
dbutils.widgets.text("run_date", "")
run_date = dbutils.widgets.get("run_date")
if not run_date:
    raise ValueError("run_date parameter is required (yyyy-MM-dd)")

# -----------------------------
# KEY VAULT SECRETS (SP for ADLS OAuth)
# -----------------------------
scope = "kv-bankrisk01-scope"
tenant_id = dbutils.secrets.get(scope=scope, key="tenant-id")
client_id = dbutils.secrets.get(scope=scope, key="sp-client-id")
client_secret = dbutils.secrets.get(scope=scope, key="sp-client-secret")
storage_account = dbutils.secrets.get(scope=scope, key="storage-account-name")

# -----------------------------
# CONFIGURE ABFSS ACCESS USING SERVICE PRINCIPAL
# -----------------------------
spark.conf.set(f"fs.azure.account.auth.type.{storage_account}.dfs.core.windows.net", "OAuth")
spark.conf.set(f"fs.azure.account.oauth.provider.type.{storage_account}.dfs.core.windows.net",
               "org.apache.hadoop.fs.azurebfs.oauth2.ClientCredsTokenProvider")
spark.conf.set(f"fs.azure.account.oauth2.client.id.{storage_account}.dfs.core.windows.net", client_id)
spark.conf.set(f"fs.azure.account.oauth2.client.secret.{storage_account}.dfs.core.windows.net", client_secret)
spark.conf.set(f"fs.azure.account.oauth2.client.endpoint.{storage_account}.dfs.core.windows.net",
               f"https://login.microsoftonline.com/{tenant_id}/oauth2/token")

# -----------------------------
# PATHS
# -----------------------------
raw_path = f"abfss://raw@{storage_account}.dfs.core.windows.net/treasury_yield_curve/dt={run_date}/*.json"
curated_path = f"abfss://curated@{storage_account}.dfs.core.windows.net/treasury_yield_curve/"
analytics_path = f"abfss://analytics@{storage_account}.dfs.core.windows.net/bankrisk_indicators/"

print("RAW:", raw_path)
print("CURATED:", curated_path)
print("ANALYTICS:", analytics_path)

# -----------------------------
# READ RAW
# -----------------------------
raw_df = spark.read.json(raw_path)

# Many public finance APIs return data under a "data" array.
# Try to normalize it. If your JSON is already flat, this still works.
df = raw_df
if "data" in df.columns:
    df = df.select(F.explode("data").alias("d")).select("d.*")

# -----------------------------
# EXPECTED FIELDS (example)
# Adjust these to match your chosen dataset fields.
# The goal is: a date + a few numeric rates
# -----------------------------
# Common pattern: record_date + rate fields
possible_date_cols = ["record_date", "date", "as_of_date", "effective_date"]
date_col = next((c for c in possible_date_cols if c in df.columns), None)
if not date_col:
    raise ValueError(f"Could not find a date column. Available columns: {df.columns}")

# Example rate columns (edit to match your API)
# If your dataset uses different names, update these:
rate_2y_col = "bc_2year"
rate_10y_col = "bc_10year"

# If those exact names don't exist, fail fast with a helpful message
missing = [c for c in [rate_2y_col, rate_10y_col] if c not in df.columns]
if missing:
    raise ValueError(
        f"Missing expected columns: {missing}. "
        f"Update rate_2y_col/rate_10y_col to match your dataset fields. "
        f"Available columns: {df.columns}"
    )

# -----------------------------
# CLEAN + CAST
# -----------------------------
clean_df = (
    df
    .withColumn("as_of_date", F.to_date(F.col(date_col)))
    .withColumn("rate_2y", F.col(rate_2y_col).cast("double"))
    .withColumn("rate_10y", F.col(rate_10y_col).cast("double"))
    .drop(date_col)
)

# Basic quality filter: keep valid dates and non-null rates
clean_df = clean_df.filter(
    F.col("as_of_date").isNotNull() &
    F.col("rate_2y").isNotNull() &
    F.col("rate_10y").isNotNull()
)

# -----------------------------
# SIMPLE BANK RISK INDICATORS
# -----------------------------
# 2y-10y spread: negative implies inversion risk signal (very common market-risk talking point)
analytics_df = (
    clean_df
    .withColumn("spread_2y_10y", F.col("rate_10y") - F.col("rate_2y"))
    .withColumn("is_inverted", F.when(F.col("spread_2y_10y") < 0, F.lit(True)).otherwise(F.lit(False)))
    .withColumn("run_date", F.lit(run_date))
    .select("as_of_date", "rate_2y", "rate_10y", "spread_2y_10y", "is_inverted", "run_date")
)

# -----------------------------
# WRITE CURATED (Delta)
# -----------------------------
(
    clean_df
    .write
    .format("delta")
    .mode("append")
    .save(curated_path)
)

# -----------------------------
# WRITE ANALYTICS (Parquet for easy Synapse querying)
# -----------------------------
(
    analytics_df
    .repartition(1)  # small dataset: keep it simple
    .write
    .mode("overwrite")
    .parquet(f"{analytics_path}dt={run_date}/")
)

print("Wrote curated Delta and analytics Parquet successfully.")
```

---

# Step 11 — Create Azure Synapse (Serverless SQL only)
Azure Portal:
1. Search **Azure Synapse Analytics** → **Create**
2. Resource group: `rg-bankrisk01`
3. Workspace name: `syn-bankrisk01<unique>`
4. Region: same
5. Create a storage account link:
   - Use the same ADLS Gen2 account
6. Create

## In Synapse Studio
Open Synapse Studio → use **SQL scripts** (Serverless SQL pool is included by default)

### Query the analytics Parquet directly
Replace `<storage>` with your storage account name.

```sql
-- Query Parquet written by Databricks (serverless, pay-per-query)
SELECT TOP 50 *
FROM OPENROWSET(
    BULK 'https://<storage>.dfs.core.windows.net/analytics/bankrisk_indicators/dt=*/',
    FORMAT = 'PARQUET'
) AS rows
ORDER BY as_of_date DESC;
```

### Example “risk” query: count inversion days
```sql
SELECT
    SUM(CASE WHEN is_inverted = 1 THEN 1 ELSE 0 END) AS inverted_days,
    COUNT(1) AS total_days
FROM OPENROWSET(
    BULK 'https://<storage>.dfs.core.windows.net/analytics/bankrisk_indicators/dt=*/',
    FORMAT = 'PARQUET'
) AS rows;
```

---

# Step 12 — Validation and basic monitoring (simple)
You can keep this simple and still look professional:

## ADF checks (cheap)
- Add a **Get Metadata** activity after the Copy to confirm the file exists in ADLS
- Add an **If Condition**: if missing, fail the pipeline
- Add another check after Databricks: verify analytics folder exists

## Logging
Write a tiny JSON log file to `logs/` per run:
- pipeline name
- run_date
- success/failure
- record counts (optional if you add a small notebook step to output counts)

---

# Step 13 — What to say on your resume (copy/paste bullets)
- Built an Azure data pipeline (**ADLS Gen2 + ADF + Databricks + Synapse Serverless**) to ingest and transform market-rate data into analytics-ready datasets.
- Implemented secure secret management with **Azure Key Vault** and eliminated hardcoded credentials using **Key Vault-backed Databricks secret scopes**.
- Orchestrated API ingestion and Spark-based transformations using **ADF pipelines** triggering **Databricks notebooks**.
- Produced curated Delta datasets and analytics Parquet outputs in a lakehouse-style layout and enabled ad-hoc querying via **Synapse Serverless SQL**.

---

# Troubleshooting (common pain points)
## “Databricks can’t access ADLS”
- Confirm the service principal has **Storage Blob Data Contributor** on the storage account
- Confirm the Key Vault secrets are correct
- Confirm the notebook spark.conf values use the correct storage account name

## “ADF can’t read Key Vault”
- Confirm ADF managed identity has **Key Vault Secrets User** role on the Key Vault

## “Synapse OPENROWSET can’t read files”
- Check storage firewall settings
- Confirm the path is correct and files exist
- Keep the analytics output as Parquet (Synapse reads it cleanly)

---

# Simple extension ideas (only if you want)
- Add a second dataset ingest (e.g., another Treasury dataset) and join in Databricks
- Add a rolling 7-day average and daily change columns
- Add a tiny “data quality” section in the notebook:
  - null rate checks
  - duplicate date checks
  - outlier checks (basic thresholds)

---

# Done
You now have a complete, production-style Azure pipeline that’s small, cheap, and realistic.

Project name: **bankrisk01**
