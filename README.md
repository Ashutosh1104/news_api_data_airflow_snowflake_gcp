# news_api_data_airflow_snowflake_gcp

News API pipeline using Airflow (Cloud Composer), Google Cloud Storage, and Snowflake. DAGs in this repo are deployed to Composer automatically with GitHub Actions.

## Repository structure

```
.github/workflows/deploy-dags.yml   # CI/CD workflow
airflow_dags22/                     # Airflow DAGs
```

## Pipeline overview

The `newsapi_data_processing_dag` DAG runs daily without backfilling past runs. Its tasks execute in this order:

1. **Fetch news:** The `newsapi_data_to_gcs` task reads the NewsAPI key from the Airflow Variable `NEWS_API_KEY`, requests English articles matching `technology` from the previous day, cleans the article fields, and saves the results as a timestamped Parquet file.
2. **Upload to Google Cloud Storage:** The Parquet file is uploaded to `gs://snowflake-projects-test-gds/news_data_analysis/`. The local copy is removed after a successful upload.
3. **Create the Snowflake table:** The `snowflake_create_table` task infers the table schema from files in the configured external stage and creates `news_api.PUBLIC.news_api_data` if it does not already exist.
4. **Load the data:** The `snowflake_copy_from_stage` task copies staged Parquet data into `news_api.PUBLIC.news_api_data`, matching columns without regard to case.
5. **Build summaries:** After the load succeeds, two tasks run in parallel. `summary_news` contains article counts and date ranges by source; `author_activity` contains article counts and distinct source counts by author.

The fetcher currently stops after the first NewsAPI page, even though it contains pagination logic. The DAG has no retries configured, so a task failure must be addressed and rerun.

## Prerequisites

- Google Cloud project with a Cloud Composer environment (`airflow-de22`, `us-central1`)
- A Snowflake account
- Admin access to this GitHub repo

## Setup

### 1. Google Cloud

1. Enable the Cloud Composer and IAM Service Account Credentials APIs.
2. Create the Composer environment with a service account that has the **Composer Worker** role.
3. Create a `github-deployer` service account with:
   - **Storage Object Admin** on the Composer DAG bucket
   - A role that allows `composer.environments.get`
4. Create a Workload Identity Pool (`github-pool`) and OIDC provider (`github-provider`):
   - Issuer: `https://token.actions.githubusercontent.com`
   - Condition: `assertion.repository == 'Ashutosh1104/news_api_data_airflow_snowflake_gcp'`
5. Allow the repository to impersonate `github-deployer` (**Workload Identity User** role).

### 2. GitHub

Add these repository secrets (Settings > Secrets and variables > Actions):

| Secret | Value |
|---|---|
| `WIF_PROVIDER` | `projects/<PROJECT_NUMBER>/locations/global/workloadIdentityPools/github-pool/providers/github-provider` |
| `GCP_SA_EMAIL` | `github-deployer@<PROJECT_ID>.iam.gserviceaccount.com` |

Optional repository variables: `COMPOSER_ENV`, `COMPOSER_REGION` (defaults are set in the workflow).

### 3. Composer environment

- In the Composer environment's PyPI packages configuration, install `apache-airflow-providers-snowflake`.
- In the Airflow UI for Composer, open **Admin > Connections** and create the Snowflake connection with ID `snowflake_conn`, which is the ID used by the DAG. Do not commit credentials to the repo.
- To store Airflow connection credentials in Google Secret Manager, configure Secret Manager as the Airflow secrets backend and enable billing for the Google Cloud project. A free-tier account without billing enabled cannot use this setup; use Composer's Airflow Connections UI if you are not configuring Secret Manager.

## Usage

1. Add or edit DAGs in `airflow_dags22/`.
2. Push to `main`.
3. The **Deploy DAGs** workflow syncs the folder to Composer and removes files in the remote DAG folder that are not in the repository folder.
4. DAGs appear in the Airflow UI within a couple of minutes.

To deploy manually: GitHub > Actions > Deploy DAGs > Run workflow.

## Recreating the Composer environment

A new environment gets a new DAG bucket. Re-grant **Storage Object Admin** to `github-deployer` on the new bucket, reinstall PyPI packages, recreate the Snowflake connection, then run the workflow manually.

## Troubleshooting

| Problem | Check |
|---|---|
| Workflow does not run | File is at `.github/workflows/`, and the push changed `airflow_dags22/` |
| Authentication fails | `WIF_PROVIDER` uses the project number; repo name in the condition matches exactly |
| Sync fails | `github-deployer` has access to the DAG bucket |
| Broken DAG in Airflow | Open the error in the UI; usually a missing PyPI package |
| Snowflake task fails | Confirm the `snowflake_conn` Airflow connection, external stage, file format, and Snowflake permissions |