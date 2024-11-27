# BQML Tutorial

This BQML tutorial has a pre-requisite of completing the Dataform tutorial (the [readme.md](https://github.com/shan-alexander/dataform-playground-boilerplate) file in this repo), in which you'll create the tables and views on which our BQML will depend.

[Pre-requisite Dataform Tutorial](https://github.com/shan-alexander/dataform-playground-boilerplate)

Alternatively, you can run the create_commodities_tables.sqlx in this repo and _____ file to skip the Dataform tutorial and jump straight into BQML.

## Introduction to BQML

BQML stands for BigQuery Machine Learning (BigQuery ML, or BQML for short). The Google documentation for an intro to BQML is here: [Introduction to AI and ML in BigQuery](https://cloud.google.com/bigquery/docs/bqml-introduction)

In this tutorial, we'll train ML models using only SQL, no Python or other steps. The data will come from our commodities and inflation tables and views, and as a bonus, we'll do it all from Dataform which adds technical excellence but also adds a few nuances.

## Step 1: Write a model for training

