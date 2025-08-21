# Newton's GCP Tutorials

This repo contains a collection of tutorials, code snippets, and screenshots to help you bootstrap a new GCP project, become familiar with Dataform, and exercise the skills used in GCP data engineering.

This repo was originally created to provide Dataform training materials for newly onboarded engineers (at a company I worked for previously, Google).

> [!NOTE]
> All of these tutorials are written by hand without any content from LLMs.

## Walkthroughs

1. **[GCP Starter](./GCP_STARTER.md)**: Kickstart a new GCP project (free trial) and configure IAM for Dataform.
2. **[Dataform](./DATAFORM.md)**: Learn the basics and best practices of Dataform.
3. **[BQML](BQML.md)**: Learn the basics of BigQuery Machine Learning, using Dataform!
4. **[Cloud Functions](CLOUD_FUNCTIONS.md)**: Learn to ingest data from 3rd party APIs, using Go, for initial backfills and recurring incremental updates, with Cloud Functions.
5. **[Cloud Composer / Airflow](COMPOSER_AIRFLOW.md)**: Learn to set up an airflow environment in Cloud Composer and write a DAG with several tasks to trigger the Cloud Function.
