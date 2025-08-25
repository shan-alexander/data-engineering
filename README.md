# Newton's GCP Tutorials

This repo contains a collection of tutorials, code snippets, and screenshots to help you bootstrap a new GCP project, become familiar with Dataform, and exercise the skills used in GCP data engineering & analysis.

This repo was originally created to provide Dataform training materials for newly onboarded engineers, in 2024. I've updated the walkthrough in summer of 2025, and still adding polishes when time allows. Eventually, I'll create some youtube video tutorials of this repo, guiding the audience through the process.

> [!NOTE]
> All of these tutorials are written by hand. 95%+ of the tutorial was written without assistance from any LLM, but in late 2025 I've been polishing and adding tips, best practices, and enhancing code snippets based on info I've gathered from LLM discussions.

## Walkthroughs

1. **[GCP Starter](./GCP_STARTER.md)**: Kickstart a new GCP project (free trial) and configure IAM for Dataform.
2. **[Dataform](./DATAFORM.md)**: Learn the basics and best practices of Dataform.
3. **[BQML](BQML.md)**: Learn the basics of BigQuery Machine Learning, using Dataform!
4. **[Cloud Functions](CLOUD_FUNCTIONS.md)**: Learn to ingest data from 3rd party APIs, using Go, for initial backfills and recurring incremental updates, with Cloud Functions.
5. **[Cloud Composer / Airflow](COMPOSER_AIRFLOW.md)**: Learn to set up an airflow environment in Cloud Composer and write a DAG with several tasks to trigger the Cloud Function.

## To Do

- Cloud Composer (Airflow) vs Cloud Scheduler
- Data Catalog, Dataplex, & Data Governance
- Taking BQML to the next level with Tensorflow custom-trained models
- Securing your GCP project with VPC configuration
- Deep-dive on Cloud Storage for data engineering purposes
- Pub/sub, Kafka, Dataflow, & Dataproc
