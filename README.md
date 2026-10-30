# news_api_data_airflow_snowflake_gcp

News API pipeline using Airflow (Cloud Composer), Google Cloud Storage, and Snowflake. DAGs in this repo are deployed to Composer automatically with GitHub Actions.

## Repository structure

```
.github/workflows/deploy-dags.yml   # CI/CD workflow
airflow_dags22/                     # Airflow DAGs
```

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

- Install the PyPI package `apache-airflow-providers-snowflake`.
- Create the Airflow connection `snowflake_default`. Do not commit credentials to the repo.

## Usage

1. Add or edit DAGs in `airflow_dags22/`.
2. Push to `main`.
3. The **Deploy DAGs** workflow syncs the folder to Composer.
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