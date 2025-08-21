# Newton's Dataform tutorial

**Pre-requisites to the steps below**:
- Have a Google Cloud Platform project already, or open a free trial (they provide $300 in credits. This tutorial won't burn more than $5 of the $300 free trial credits).
- Have sufficient IAM permissions. If you are not creating a new GCP project (ie you are working in an existing GCP project of which you do not have owner permissions), you will need the `roles/dataform.editor` role on your GCP user (do this in IAM, or ask someone in your GCP project to grant this role to you).
- The Dataform service account also needs the correct IAM roles. If you haven't enabled Dataform in GCP and provided IAM permissions to the Dataform service account already, follow the steps in the [GCP_STARTER.md](./GCP_STARTER.md) walkthrough first, and then come back to this page.

## Step 1: Create and Structure your Dataform Repo
- Navigate to BigQuery > Dataform
- Create a new dataform repo
- Create a workspace
- Initialize the workspace
- Give it the folder structure as outlined here: https://cloud.google.com/dataform/docs/structure-repositories (to put it briefly, create 4 folders inside the definitions folder: sources, intermediate, outputs, extra).
- Make sure you have a workflow_settings.yaml file configured

My yaml file looks like this:

###### workflow_settings.yaml
```sql
defaultProject: dataform-intro-469416
defaultLocation: US
defaultDataset: dataform_playground
defaultAssertionDataset: dataform_assertions
dataformCoreVersion: 3.0.0
```

***

## Step 2: Create new tables from Dataform


I've created a dataset called `dataform_playground` and want to create a new table `radio_spelling_alphabet`.
I put this code below in a file called `create_radio_alphabet.sqlx` in the `/extra` folder.

###### definitions/extra/create_radio_alphabet.sqlx
```sql
config {
    type: "operations"
}


CREATE OR REPLACE TABLE dataform_playground.radio_alphabet AS
SELECT * FROM UNNEST([
  STRUCT(NULL AS num, '' AS letter, '' as code),
  (1,  'a', 'Alpha'),
  (2,  'b', 'Bravo'),
  (3,  'c', 'Charlie'),
  (4,  'd', 'Delta'),
  (5,  'e', 'Echo'),
  (6,  'f', 'Foxtrot'),
  (7,  'g', 'Golf'),
  (8,  'h', 'Hotel'),
  (9,  'i', 'India'),
  (10, 'j',  'Juliett'),
  (11, 'k',  'Kilo'),
  (12, 'l',  'Lima'),
  (13, 'm',  'Mike'),
  (14, 'n',  'November'),
  (15, 'o',  'Oscar'),
  (16, 'p',  'Papa'),
  (17, 'q',  'Quebec'),
  (18, 'r',  'Romeo'),
  (19, 's',  'Sierra'),
  (20, 't',  'Tango'),
  (21, 'u',  'Uniform'),
  (22, 'v',  'Victor'),
  (23, 'w',  'Whiskey'),
  (24, 'x',  'Xray'),
  (25, 'y',  'Yankee'),
  (26, 'z',  'Zulu')
])
WHERE NOT num IS NULL
```
When you run this file, because it is a `type: "operations"`, it will execute the SQL as-is and create the table.

Chances are, you haven't yet created the dataset `dataform_playground` though, so go ahead and create that now, either via the BQ console or with this SQL:
```sql
CREATE SCHEMA IF NOT EXISTS `project_id.dataset_name`
OPTIONS(
  location = 'US' -- your workflow_settings.yaml location needs to match the location you set here
);
```
You can run the above SQL snippet from BQ or Dataform, or create the dataset via point-and-click in the BigQuery Studio UI. Then re-run the `create_radio_alphabet.sqlx` so that the table is successfully created.


Now create a file to see the results of that table. Let's create that file in the `/intermediate` folder and call the file `radio_alphabet_consonants.sqlx`.

###### definitions/intermediate/radio_alphabet_consonants.sqlx
```sql
config {
    type: "view"
}

SELECT
*
FROM dataform_playground.radio_alphabet
WHERE num NOT IN (1,5,9,15,21)
ORDER BY num
```

This ought to show you the results, without the vowels rows. However, in Dataform we want to use a special feature to refer to tables in our query. Instead of explicitly writing `FROM dataform_playground.radio_alphabet` we instead want to write:

`FROM ${ref("radio_alphabet")}`

Change that in your file now, and run it.

###### definitions/intermediate/radio_alphabet_consonants.sqlx
```sql
config {
    type: "view"
}

SELECT
*
FROM ${ref("radio_alphabet")}
WHERE num NOT IN (1,5,9,15,21)
ORDER BY num
```

You'll likely get the error message `Could not resolve "radio_alphabet"`. The Dataform compiler is unable to reference `radio_alphabet` because we've not yet declared this table in our `/sources` folder. Let's do that now -- create a file in the sources folder with the exact same name as the bigquery table (the table we created is called `radio_alphabet` so your file should be named `radio_alphabet.sqlx`).

Give the file the needed declaration:
```sql
config {
  type: "declaration",
  database: "your_project_id_here",
  schema: "dataform_playground",
  name: "radio_alphabet",
}
```

Now go back to `/intermediate/radio_alphabet_consonants.sqlx` and run it. Notice how the "Compiled Queries" tab (on the righthand side of the editor) shows the explicit FROM clause. The ${ref()} function is changed by the Dataform compiler to express the explicit reference.

![image of compiled query failed to load](img/dataform_compiled_queries.jpeg "Screenshot of compiled query")

And the results...

![image of results failed to load](img/radio_alphabet_consonants_results.png "Screenshot of the query results")

Using the ${ref()} feature is something we always want to do in Dataform, as a best practice.

Now that you've made a few files and your repo is compiling successfully, go ahead and commit your changes. Give a commit message (ideally in the present tense) and push.



***

## Step 3: Add a few more data tables

I've prepared a handful of datasets we can use for experimentation with Dataform, BQML, and Pipe Syntax.

Use the `create_commodities_tables.sqlx` file I've provided in the repo. Copy and paste this into a Dataform file `definitions/extra/create_commodities_tables.sqlx`, and run it. It's too long of a query to put here, hence the separate file. This will give you REAL data with a handful of tables with which you can tinker, analyze, create BQML forecasts, and so on.

***

## Step 4: Declare the tables so they can be referenced via `${ref()}`

Make a new file for the `oil` data table. Note that the filename must match the table name exactly. Declaration files have only a config block with a few lines:

###### /definitions/sources/oil.sqlx
```sql
config {
  type: "declaration",
  database: "tokyo-hold-441414-t2",
  schema: "dataform_playground",
  name: "oil",
}
```

Now click the three dots on this file from the Files Pane and select `Duplicate` and give this new file the name `/definitions/sources/gold.sqlx` and change line 5 to `name: "gold"`.

Do this step again for the `inflation` data table, and all the other tables you have created. It's a few minutes of tediousness, but required.

***

## Step 5: Make a view to union the data tables

Each data table has three columns and can be unioned to create a view (which you might consider to pipe into a Looker Explore if your project uses Looker), so let's create that view. I'm deciding to put this view in the `/intermediate` folder, because I consider it a "silver" tier view per medallion architecture ideology. I'll explain this more later. Create the file and fill it with the SQL.

###### definitions/intermediate/commodities_and_inflation.sqlx
```sql
config {
    type: "view"
}

SELECT
month_,
index as price,
inflation as percent_change,
"index" as price_type,
"inflation" as data_type
from ${ref("inflation")}

union all

SELECT
month_,
price_per_troyounce as price,
percent_change,
"troy ounce" as price_type,
"gold" as data_type
from ${ref("gold")}

union all

SELECT
month_,
price_per_metricton as price,
percent_change,
"troy ounce" as price_type,
"silver" as data_type
from ${ref("silver")}

union all

SELECT
month_,
price_per_metricton as price,
percent_change,
"metric ton" as price_type,
"copper" as data_type
from ${ref("copper")}

union all

SELECT
month_,
price_per_barrel as price,
percent_change,
"barrel" as price_type,
"oil" as data_type
from ${ref("oil")}

union all

SELECT
month_,
price_per_gallon as price,
percent_change,
"gallon" as price_type,
"gas" as data_type
from ${ref("gas")}

union all

SELECT
month_,
price_per_kilo as price,
percent_change,
"kilo" as price_type,
"coffee" as data_type
from ${ref("coffee")}

union all

SELECT
month_,
price_per_kilo as price,
percent_change,
"kilo" as price_type,
"beef" as data_type
from ${ref("beef")}

union all

SELECT
month_,
price_per_kilo as price,
percent_change,
"kilo" as price_type,
"bananas" as data_type
from ${ref("bananas")}

union all

SELECT
month_,
price_per_kilo as price,
percent_change,
"metric ton" as price_type,
"rice" as data_type
from ${ref("rice")}
```

Go ahead and run it, to see the output results.

![image of results failed to load](img/commodities_and_inflation.png "Screenshot of the query results")

Now let's make another view to `SELECT *` from the view we just made. We'll put the new view in the ouputs folder (as a gold tier query, per medallion architecture).

###### definitions/output/v_commodities_and_inflation.sqlx
```sql
config {
    type: "view"
}

select * from ${ref("commodities_and_inflation")}
```

Notice that we get an error.

![image failed to load](img/commodities_and_inflation_not_found.png "Screenshot of the error message")

The ${ref()} was unable to find the table we made in the `/intermediate` folder. Why? Because we've not yet executed the file, we've only written the file in the Dataform repo but not actually executed code to state `create or replace view`. So let's do that now.

Click on the `/intermediate/commodities_and_inflation.sqlx` file and click `START EXECUTION` > `Actions` > `commodities_and_inflation`. Click the button at the bottom of the popup `Start Execution`.

![image failed to load](img/commodities_and_inflation_execution.png "Screenshot of the execution")

At the bottom of the console, you should see a small popup saying `Successfully created workflow execution - DETAILS` and you can click the DETAILS word to see what happened. Do that now.

![image failed to load](img/successful_execute.png "Screenshot of the success message")

It shows you the code that Dataform executed for you (via the Dataform service account) and around line 24 you see it runs `create or replace view`.

![image failed to load](img/executed_code.png "Screenshot of the executed code")

Go back into your workspace and run again the file `/output/v_commodities_and_inflation.sqlx`. It should run without errors this time, because it was able to find the referenced view (because we've now created the view by executing the dataform file').

You might also ask, why did we name the output file with a `v_` prefix? We did this as a convention to denote this is a view, and specifically an `/ouput` folder view. When browsing through BigQuery, we won't be able to see which views are `/intermediate` and which are `/ouput`. If we follow this prefix convention, it will make our BigQuery datasets easier to troubleshoot and navigate. Using `select *` as a gold-tier query is a standard practice in data engineering so that other views, models, Looker Explores, etc., can point to the gold-tier query reliably, even when the underlying data sources and views are regularly changing. This reduces the technical burden during migrations, new 3rd party datasets being weaved in, architectural changes, etc. Your project leader might use a different convention. Check with your project leader to see what conventions are being used. Note that our intermediate script (full of unions) could also create a table, instead of a view, by changing the `type` in the config block.

Your files pane now ought to look something like this:

![image failed to load](img/files_pane.png "Screenshot of the files pane")

And from BigQuery, it ought to look similar to this:

![image failed to load](img/bigquery_pane.png "Screenshot of the files pane")

***

## Step 6: Do a quick exploration of the data to look for insights

Write a view to isolate gold, rice, and bananas and chart their price as separate lines on a line chart. Consider yourself an investor in mid-year 1996, and you invested in one of these three commodities, with the intention of selling your position in 10 years (mid-2006). Which commodity would have higher returns?

Go ahead and write this from scratch in the /extra folder as a view. We're just going to use the BigQuery charting tool, so we'll need to pivot on the commodities.

###### definitions/extra/gold_or_rice_or_bananas.sqlx
```sql
config {
    type: "view"
}

with some_filters as (
select
cast(month_ as date) as month_,
data_type,
-- convert price per kilo to metric ton for bananas, to compare with rice metric ton and gold troy ounce
sum(case when data_type = "bananas" then price * 1000 else price end) as price,
from ${ref("v_commodities_and_inflation")}
where data_type in ("bananas", "rice", "gold")
and month_ > "1994-12-01"
group by all
)

SELECT * FROM some_filters
PIVOT(
sum(price) FOR data_type IN ("bananas", "rice", "gold")
)
order by month_ asc
```

Now run it. Then click the three dots in the top right of the results pane in Dataform, and select `View job in SQL workspace`.


![image failed to load](img/view_in_sql_workspace.png "Screenshot of the SQL Workspace navigation")

This will open BigQuery in a new tab, and show the same results. We can then click the `Chart` tab (it doesn't seem to work in Dataform, but the Chart tab works from BigQuery). Typically we'll use Looker or another dashboarding tool for charting, but sometimes in the data exploration phase, the BigQuery charting feature can be useful for simple explorations like this one.

![image failed to load](img/gold_rice_bananas_chart.png "Screenshot of the chart")

You should see three lines, which begin around the same price in the 1990s, but begin separating widely around 2008.

***

## Step 7: Find commodities whose price is correlated

Now that we've seen the data visualized, let's find out if any of the commodities are statistically correlated.

###### definitions/extra/commodity_correlation_example.sqlx
```sql
config {
    type: "view"
}

select
corr(gold, bananas) as gold_banana,
corr(gold, rice) as gold_rice,
corr(bananas, rice) as banana_rice
from ${ref("gold_or_rice_or_bananas")}
```

| gold_banana | gold_rice | banana_rice |
| :------------------- | :----------: | ----------: |
| 	0.917   	      | 0.764     | 0.682      |


You can see from the output, that gold and bananas are tightly correlated, moreso than gold & rice and moreso than banana & rice.

Since we've found a good correlation, we can use the price of gold to forecast the price of bananas, and vice-versa, using BQML's arima_xreg model. Arima models are usually simple calculations, but the XREG model of arima can use multiple variables (so long as they are correlated) to make forecasts.  Therefore, it's important to find out which data points are correlated and which are not.

To learn more about BQML, see the [BQML Readme](BQML.md ./BQML.md) readme walkthrough (still a work-in-progress).


To be thorough, let's look at other correlation possibilities.

###### definitions/extra/commodity_correlation.sqlx
```sql
config {
    type: "view"
}

with pivot_setup as (
select
cast(month_ as date) as month_,
data_type,
price,
from `tokyo-hold-441414-t2.dataform_poc.v_commodities_and_inflation`
where month_ > "1994-12-01"
)

, pivoted_commodities as (
SELECT * FROM pivot_setup
PIVOT(
sum(price) FOR data_type IN ("bananas", "rice", "gold","silver","copper","oil","diesel","gas","beef","coffee","inflation")
)
order by month_ asc
)

select
corr(gold, bananas) as gold_banana,
corr(gold, rice) as gold_rice,
corr(gold, coffee) as gold_coffee,
corr(gold, beef) as gold_beef,
corr(gold, oil) as gold_oil,
corr(gold, gas) as gold_gas,
corr(gold, diesel) as gold_diesel,
corr(gold, copper) as gold_copper,
corr(gold, silver) as gold_silver,
corr(gold, inflation) as gold_inflation,

corr(oil, bananas) as 	oil_banana,
corr(oil, rice) as 		oil_rice,
corr(oil, coffee) as 	oil_coffee,
corr(oil, beef) as 		oil_beef,
corr(oil, gas) as 		oil_gas,
corr(oil, diesel) as 	oil_diesel,
corr(oil, copper) as 	oil_copper,
corr(oil, silver) as 	oil_silver,
corr(oil, inflation) as oil_inflation
from pivoted_commodities
```


To do:
- continue basics walkthrough
- BQML forecast arima_xreg & arima_plus
- Pipe Syntax
- Cloud Composer
