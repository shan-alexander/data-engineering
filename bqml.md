# Newton's BQML Tutorial

This BQML tutorial has a pre-requisite of completing the Dataform tutorial (the [readme.md](https://github.com/shan-alexander/dataform-playground-boilerplate?tab=readme-ov-file#dataform-playground-boilerplate) file in this repo), in which you'll create the tables and views on which our BQML will depend.

[Pre-requisite Dataform Tutorial](https://github.com/shan-alexander/dataform-playground-boilerplate?tab=readme-ov-file#dataform-playground-boilerplate)

Alternatively, you can run the create_commodities_tables.sqlx in this repo and _____ file to skip the Dataform tutorial and jump straight into BQML.

## Introduction to BQML

BQML stands for BigQuery Machine Learning (BigQuery ML, or BQML for short). The Google documentation for an intro to BQML is here: [Introduction to AI and ML in BigQuery](https://cloud.google.com/bigquery/docs/bqml-introduction)

In this tutorial, we'll train ML models using only SQL, no Python or other steps. The data will come from our commodities and inflation tables and views, and as a bonus, we'll do it all from Dataform which adds technical excellence but also adds a few nuances.

There's a decent [cheatsheet of BQML syntax on Medium](https://medium.com/fifty-five-data-science/bigquery-machine-learning-cheat-sheet-7c053b21a657), if you'd like to supplement [the official Google Docs](https://cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-create).

## Step 1: Write an Arima Plus model

You can decide whether to create a new Dataform repo for your BQML models, which adds a few steps of tediousness but is more appropriate. For the purposes of a tutorial, however, you can also continue from your existing Dataform repo with the commodities and inflation data.

###### definitions/extra/arima_plus_bananas_price.sqlx
```sql
config {
   type: "operations",
   description: "Trains an arima model to predict the monthly price of bananas."
}


create or replace model dataform_playground.arima_plus_bananas_price


options ( 
 MODEL_TYPE='ARIMA_PLUS',
 TIME_SERIES_TIMESTAMP_COL='month_',
 TIME_SERIES_DATA_COL='price'
 )
 as (training_data as (
 with historical_data as (
 select
   cast(month_ as date) as month_
 , sum(price) as price
 from ${ref("v_commodities_and_inflation")} 
 where month_ between '1994-12-01' and '2022-12-01'
 and data_type = 'bananas'
 group by 1
 )
 select * from historical_data order by month_ asc)
);
```

This ought to successfully train the model.

![image failed to load](img/create_model_success.png "Screenshot of the create model success message")

Notice in BigQuery you now have a model available:

![image failed to load](img/show_model_in_bq.png "Screenshot of the model in BigQuery")

## Step 2: Use the Arima Plus model to forecast



## Step 3: Create an Arima XREG model


## Step 4: Use the Arima XREG model to forecast

