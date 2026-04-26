# Getting Started with ABINZ Orbit on Snowflake

**Duration:** 30 minutes  
**Level:** Beginner  
**Prerequisites:** Snowflake account with ACCOUNTADMIN access

---

## Overview

ABINZ Orbit is a **Snowflake Native App** that turns your Snowflake tables into a governed, AI-ready semantic layer.

It automatically profiles your data, discovers relationships between tables, and generates a Cortex Analyst model — so your team can ask questions in plain English and get SQL-backed answers instantly.

Everything runs entirely inside your Snowflake account using the Native Apps framework. Your data never leaves, and no external services or additional infrastructure are required.

In most environments, you can go from raw tables to a working AI query interface in under an hour.

**What you will build:**

In this quickstart you will:

1. Install ABINZ Orbit from the Snowflake Marketplace
2. Grant access to your data
3. Create a workspace and run metadata analysis
4. Query your data in plain English using the AI Assistant (with built-in Self-Healing SQL for automatic error recovery)
5. Set up a KPI Outcome Contract to track business results

---

## What You Will Need

- A Snowflake account (any cloud, any region)
- ACCOUNTADMIN role (for installation and initial grants)
- At least one database with tables you want to analyze
- A running warehouse (e.g., `COMPUTE_WH`)

> **Note:** No sample data is required. ABINZ Orbit works with tables already in your Snowflake account. If you want to follow along with a known dataset, any Snowflake sample database (e.g., `SNOWFLAKE_SAMPLE_DATA`) works fine.

---

## Step 1 — Install ABINZ Orbit from the Marketplace

1. Log in to your Snowflake account at **app.snowflake.com**
2. Click **Data Products** → **Marketplace** in the left sidebar
3. Search for **ABINZ Orbit**
4. Click **Get** on the listing
5. Choose a warehouse to use for the app (e.g., `COMPUTE_WH`)
6. Click **Get** to confirm — the app installs in under a minute

Once installed, navigate to **Data Products** → **Apps** → **ABINZ Orbit** to open it.

---

## Step 2 — Grant the App Access to Your Data

ABINZ Orbit can only read tables you explicitly grant it access to. Run the following as **ACCOUNTADMIN**:

```sql
-- Grant access to a specific database and schema
GRANT USAGE ON DATABASE <YOUR_DATABASE> TO APPLICATION ABINZ_ORBIT;
GRANT USAGE ON SCHEMA <YOUR_DATABASE>.<YOUR_SCHEMA> TO APPLICATION ABINZ_ORBIT;
GRANT SELECT ON ALL TABLES IN SCHEMA <YOUR_DATABASE>.<YOUR_SCHEMA> TO APPLICATION ABINZ_ORBIT;

-- (Optional) Include future tables automatically
GRANT SELECT ON FUTURE TABLES IN SCHEMA <YOUR_DATABASE>.<YOUR_SCHEMA> TO APPLICATION ABINZ_ORBIT;
```

Replace `<YOUR_DATABASE>` and `<YOUR_SCHEMA>` with your actual values. For example, using Snowflake's sample data:

```sql
GRANT USAGE ON DATABASE SNOWFLAKE_SAMPLE_DATA TO APPLICATION ABINZ_ORBIT;
GRANT USAGE ON SCHEMA SNOWFLAKE_SAMPLE_DATA.TPCH_SF1 TO APPLICATION ABINZ_ORBIT;
GRANT SELECT ON ALL TABLES IN SCHEMA SNOWFLAKE_SAMPLE_DATA.TPCH_SF1 TO APPLICATION ABINZ_ORBIT;
```

---

## Step 3 — Grant App Roles to Your Users

ABINZ Orbit has two roles:

| Role | Who gets it | Can do |
|---|---|---|
| `ABINZ_APP_ADMIN_ROLE` | You (the admin) | Everything: create/delete workspaces, start workers, manage settings |
| `ABINZ_APP_ANALYST_ROLE` | Your team members | Create workspaces, run analysis, ask AI questions |

Grant access by calling the built-in stored procedure:

```sql
-- Grant admin access (run as ACCOUNTADMIN)
CALL ABINZ_ORBIT.ABINZ_KG.GRANT_ANALYST_ACCESS_TO_USER('<YOUR_SNOWFLAKE_USERNAME>');
```

The procedure grants analyst access to the specified user. To assign admin-level access, grant the `ABINZ_APP_ADMIN_ROLE` application role to an account role (run as ACCOUNTADMIN):

```sql
-- Grant the admin application role to an account role
GRANT APPLICATION ROLE ABINZ_ORBIT.ABINZ_APP_ADMIN_ROLE TO ROLE <ACCOUNT_ROLE_NAME>;
```

Then ensure the target user has that account role granted to them in Snowflake.

---

## Step 4 — Start the Background Worker

ABINZ Orbit uses a **Serverless Task** (no warehouse needed) to process metadata in the background. This is a **one-time setup** — run it once as admin so it is ready when analysis begins in Step 7:

```sql
CALL ABINZ_ORBIT.ABINZ_KG.START_BACKGROUND_WORKERS();
```

This creates a task that runs every 60 seconds to process analysis jobs in the background. It automatically stops when there is no work to do.

You can verify the worker is running:

```sql
SHOW TASKS IN APPLICATION ABINZ_ORBIT;
```

A healthy output shows the task with `state = started` and a recent `scheduled_time` timestamp. If state shows `suspended`, re-run the `START_BACKGROUND_WORKERS` call.

---

## Step 5 — Create a Workspace

> **Role note:** From this step onward, ACCOUNTADMIN is no longer required. Proceed as a user with `ABINZ_APP_ADMIN_ROLE` or `ABINZ_APP_ANALYST_ROLE`.

A workspace is an isolated project that holds your analysis for a specific database and schema combination.

1. Open ABINZ Orbit — you land on the **Home** module
2. Click **Create New Workspace**
3. Name it something descriptive, e.g., `Claims Q1 2026`
4. Click **Create**

Your workspace is now active. All analysis, relationships, and AI queries in this workspace are isolated from other workspaces.

---

## Step 6 — Connect Your Data Source

1. Go to the **Data Sources** module (left sidebar)
2. Click **Browse Databases**
3. Select the database and schema you granted access to in Step 2
4. Select the tables you want to include — you can pick individual tables or include all
5. Click **Add to Workspace**

Once you add tables, ABINZ Orbit automatically starts analyzing them in the background.

---

## Step 7 — Run Metadata Analysis

The background worker (started in Step 4) processes your tables automatically. For each table it:

- Counts rows and columns
- Profiles each column: data type, cardinality, null rate, sample values
- Flags potential PII columns (e.g., SSN, DOB, email) based on column names and naming patterns — flagging is **informational only**; it marks the column in the UI for human review and does not restrict access or prevent analysis
- Calculates a **trust score** (0–100) based on completeness and consistency

You can watch progress in the **Data Sources** module — each table shows a status badge:

| Status | Meaning |
|---|---|
| `PENDING` | Queued, waiting for worker |
| `RUNNING` | Currently being analyzed |
| `COMPLETED` | Analysis done, ready to use |
| `FAILED` | Error — check the table for issues |

Analysis typically completes in 1–3 minutes per table, depending on table size and warehouse performance.

> **Large tables:** For tables over 10,000 rows, ABINZ Orbit automatically samples 500 rows to keep analysis fast. Full row counts are still captured.

---

## Step 8 — Discover Relationships

Once at least two tables are analyzed, go to the **Relationships** module.

Click **Discover Relationships**. ABINZ Orbit runs its **NAME_MATCH** heuristic:

- Compares column names across tables (e.g., `claim_id` in one table → `claim_id` in another)
- Checks data types and sample value overlap
- Assigns a **confidence score** from 0.0 to 1.0

High-confidence relationships are automatically accepted (default threshold: 0.7, configurable in Settings). Lower the threshold (e.g., 0.5) to surface more candidates for manual review; raise it (e.g., 0.85) to accept only the highest-certainty matches automatically. You can:

- Review and confirm suggested joins
- Add manual relationships the heuristic missed
- Remove false positives
- Adjust confidence thresholds in Settings

> **Note:** Relationship discovery typically completes in 1–2 minutes. For schemas with many tables, it may take slightly longer — this is normal.

Accepted relationships feed directly into the semantic model generation step.

---

## Step 9 — Generate a Cortex Analyst Semantic Model

Go to the **Model Preview** module and click **Generate Semantic Model**.

ABINZ Orbit builds a Cortex Analyst-compatible YAML model from your tables and confirmed relationships. The model defines:

- **Tables** with human-readable descriptions
- **Columns** with business-friendly names and data types
- **Joins** based on confirmed relationships
- **Metrics** for common aggregations (count, sum, average)

You can review and edit the YAML directly in the module before saving. The model is stored in your Snowflake account — Cortex Analyst uses it to translate plain English into SQL.

---

## Step 10 — Ask a Question in Plain English

Go to the **AI Assistant** module.

Type a question in the chat box. For example:

> "What are the top 10 procedure codes by claim count this year?"

> "Show me denial rate by payer for Q1 2026."

> "Which members had more than 5 claims in the last 90 days?"

ABINZ Orbit uses **Snowflake Cortex** to interpret your question using the semantic model:

1. Generates a SQL query based on your intent
2. Executes it inside your Snowflake account
3. Returns results in a table and natural language summary

If the generated SQL fails (e.g., due to a schema change), the **Self-Healing SQL** engine automatically detects the error, adjusts the query, and retries — without any action from you.

---

## Step 11 — (Optional) Set a KPI Outcome Contract

Beyond querying data, Orbit also lets you track whether insights translate into real business outcomes.

The **KPI Studio** module lets you define a KPI, set a target, and measure verified lift over time.

1. Go to **KPI Studio** → **Outcome Contracts**
2. Click **New Contract**
3. Define your KPI — e.g., "Claims Denial Rate"
4. Set a baseline value (e.g., 12.4%)
5. Set a target (e.g., 9%)
6. Log the intervention you are testing (e.g., "New prior auth workflow launched April 2026")
7. Click **Save Contract**

ABINZ Orbit tracks the KPI over time and shows you the measured lift — the actual change vs. your target — so you have a quantifiable record of business outcomes.

---

## Step 12 — (Optional) Explore the Metrics Library

Go to the **Metrics Library** module to find pre-defined reusable metrics. You can:

- Browse metrics by type: `SIMPLE`, `DERIVED`, `RATIO`, `CUMULATIVE`
- Add custom metrics relevant to your business
- Use metrics in AI queries by name (e.g., "Show me denial_rate by month")

---

## Summary

You have successfully:

- Installed ABINZ Orbit as a Snowflake Native App
- Connected your data without moving it outside Snowflake
- Profiled tables and detected PII automatically
- Discovered joins using AI-assisted relationship matching
- Generated a Cortex Analyst semantic model
- Queried your data in plain English
- (Optionally) set up KPI tracking with Outcome Contracts

All of this ran entirely inside your Snowflake account. No data left your environment, and no external services were required. Your existing Snowflake governance, RBAC, and encryption policies applied automatically.

---

## Next Steps

- **Add more schemas** — repeat Steps 6–9 for additional datasets in the same workspace
- **Invite team members** — run `GRANT_ANALYST_ACCESS_TO_USER` for each user
- **Explore the Data Explorer** — the **Data Explorer** module lets you browse and filter any table interactively
- **Review Observability** — the **Observability** module shows query performance, worker job history, and error logs

---

## Resources

- [ABINZ Orbit — Free Trial on Snowflake Marketplace](https://app.snowflake.com/marketplace/listing/GZU6Z1HLBPAXN/abinz-llc-abinz-orbit)
- [ABINZ Orbit Enterprise Edition](https://app.snowflake.com/marketplace/listing/GZU6Z1HLBPBCL/abinz-llc-abinz-orbit-enterprise-edition)
- [Snowflake Cortex AI Documentation](https://docs.snowflake.com/en/user-guide/snowflake-cortex/overview)
- [Snowflake Native Apps Framework](https://docs.snowflake.com/en/developer-guide/native-apps/native-apps-about)
- [Cortex Analyst Documentation](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-analyst)

---

*© 2026 ABINZ LLC. All rights reserved.*  
*ABINZ Orbit is a Snowflake Native App. Snowflake and Cortex are trademarks of Snowflake Inc.*
