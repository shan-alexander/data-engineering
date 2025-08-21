# Cloud Composer aka Airflow

This guide assumes you've already set up a Cloud Run Function, and you want to orchestrate a workflow, a DAG, to trigger that cloud function at specified intervals (daily). If you've not yet done this, visit the [Cloud Functions](CLOUD_FUNCTIONS.md) page for a walkthrough.

**Getting Started with Composer**

First, enable Cloud Composer API in GCP. Then see if you can create an environment. If you are using a free trial, unfortunately you must enable billing to create an airflow environment. You can still use the free credits for the airflow environment, but note that as of 2025 the estimated cost of an composer/airflow environment is around $400 per month. So if you set this up, remember to delete the environment when you are done, to avoid incurring charges. The $300 free tier of GCP will cover your airflow environment for more than 2 weeks, if you're only doing this tutorial. The environment runs 24/7, so there's no way to pause it.

## Step 1: Create a service account for Cloud Composer

When you create an environment in Cloud Composer, you'll want to assign it a service account (SA), ideally a specific SA dedicated only to Composer (principle of least privilege). Go to IAM > Service Accounts to create one, give it a name, and the following roles:

- roles/bigquery.user
- roles/bigquery.dataOwner
- roles/composer.worker
- roles/iam.serviceAccountUser

[image here]

## Step 2: Create a Cloud Composer Environment

From the Cloud Composer landing page, create an environment, and assign it the service account you just created.

[image here]

It will take a few minutes to prepare the environment. Once ready, enter the environment, and go to the DAGs tab, and open the DAGs folder.

[image here]

## Step 3: Upload your DAG python file

Upload a `.py` file into the DAG folder. Here is the DAG I came up with for this tutorial project:

```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.providers.http.operators.http import HttpOperator

# Define the list of cities with their coordinates
cities = [
    {"city": "New York City", "latitude": 40.7128, "longitude": -74.0060},
    {"city": "Charlotte NC", "latitude": 35.2271, "longitude": -80.8431},
    {"city": "Denver CO", "latitude": 39.7392, "longitude": -104.9903},
    {"city": "San Francisco CA", "latitude": 37.7749, "longitude": -122.4194},
    {"city": "Vancouver Canada", "latitude": 49.2827, "longitude": -123.1207},
    {"city": "Rome Italy", "latitude": 41.9028, "longitude": 12.4964},
    {"city": "Berlin Germany", "latitude": 52.5200, "longitude": 13.4050},
    {"city": "Bogota Colombia", "latitude": 4.7110, "longitude": -74.0721},
    {"city": "Lima Peru", "latitude": -12.0464, "longitude": -77.0428},
    {"city": "Copenhagen Denmark", "latitude": 55.6761, "longitude": 12.5683},
    {"city": "Moscow Russia", "latitude": 55.7558, "longitude": 37.6173},
    {"city": "Ho Chi Minh City Vietnam", "latitude": 10.7626, "longitude": 106.6602},
    {"city": "Tokyo Japan", "latitude": 35.6762, "longitude": 139.6503},
]

# Default arguments for the DAG
default_args = {
    "owner": "airflow",
    "depends_on_past": False,
    "email_on_failure": False,
    "email_on_retry": False,
    "retries": 1,
    "retry_delay": timedelta(minutes=5),
    "start_date": datetime(2025, 8, 21),  # Set to today or earlier
}

# Define the DAG
with DAG(
    dag_id="fetch_weather_data_daily",
    default_args=default_args,
    description="Triggers Cloud Run function to fetch weather data for multiple cities daily",
    schedule_interval="@daily",
    catchup=False,  # Avoid backfilling
) as dag:

    # Create an HttpOperator task for each city
    for city in cities:
        city_name = city["city"].replace(" ", "_").lower()  # Sanitize for task_id
        task = HttpOperator(
            task_id=f"fetch_weather_{city_name}",
            http_conn_id="cloud_run_weather",  # Airflow connection ID for Cloud Run
            endpoint="/",  # Root endpoint of Cloud Run function
            method="GET",
            headers={"Content-Type": "application/json"},
            data={
                "city": city["city"],
                "latitude": str(city["latitude"]),
                "longitude": str(city["longitude"]),
            },
            response_check=lambda response: response.status_code == 200,
            log_response=True,
        )

```

After a few minutes, the DAG should appear in your `Composer > Environment > DAG` tab. Click into the DAG for the "DAG Details" page. Click the "Trigger DAG" button to manually execute the DAG.

As the DAG runs, you'll see logs and success/fail messages.

[image here]

This DAG will actually fail some of the tasks because it reaches the limit of what the free API on Open Meteo will offer in a single day.

Learn more about writing DAGs for Airflow / Cloud Composer at [https://cloud.google.com/composer/docs/composer-3/write-dags](https://cloud.google.com/composer/docs/composer-3/write-dags).

## Celebrate!

That's it! You've successfully:
- Used Dataform for transforming data
- Used Dataform for creating BQML machine learning forecasts
- Created a 2nd gen Cloud Run Function using Golang to extra data from an API and ingest it into BigQuery
- Setup a Cloud Composer environment and service account
- Created a DAG and manually triggered it

There's much more to learn, in future iterations of this training pack I'll add more useful examples of Cloud Composer, which for the current tutorial is a bit of overkill (would be cheaper to use Cloud Scheduler for this use-case).
