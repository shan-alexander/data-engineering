# PyArrow Parquet for Big Data

In Google Cloud Platform, the data can be stored in a variety of ways, including:
- Standard BigQuery Table
- Cloud Storage as JSON files
- Cloud Storage as Parquet files
- Cloud Storage as Avro files

Parquet and Avro are common file types for "Big Data" (typically, a terabyte or more as of 2025), and their use includes optimizations for improved lookups (speed) and compression (size). Both file types are managed with Python, PyArrow, and Pandas.

When creating a data warehouse table from GCS, this is considered an external table, as opposed to a native BQ table. Let's compare the two.

| Feature | **BigQuery External Table** | **Standard BigQuery Table** |
| :--- | :--- | :--- |
| **Data Storage** | Data remains in GCS; BigQuery only reads the metadata. | Data is stored in BigQuery's managed, columnar storage format. |
| **Performance** | Excellent for queries that leverage partitioning/clustering. Query speed is dependent on GCS performance. | Generally faster for all query types because data is stored in BigQuery's native, highly-optimized format. |
| **Optimization** | Manual: You have to optimize the files (partitioning/sorting) yourself before creating the external table. | Automatic: BigQuery handles optimization, clustering, and data organization automatically for peak performance. You can apply clustering and partitioning via SQL. |
| **Inserts/Updates** | Not recommended for frequent inserts/updates. | Designed for high-volume inserts and updates. |
| **Cost** | You pay for GCS storage and BigQuery queries. | You pay for BigQuery storage and BigQuery queries. |

A SQL query on a BigQuery external table pointing to partitioned Parquet files can be faster than a standard BigQuery table if the standard table is not also partitioned and clustered. However, **a well-partitioned and clustered standard BigQuery table will almost always outperform an external table**, especially for complex queries that don't benefit from simple partition pruning. The key trade-off is the **manual** optimization of external files versus the managed, automatic optimization of standard BigQuery tables. In most production scenarios, the performance and management benefits of a standard BigQuery table outweigh the flexibility of an external table.

Most APIs provide data in JSON format, which is suitable for data transfer. The most inefficient data pipeline would store the JSON, and then create an external BQ table from the JSON files -- this pattern should be avoided. JSON files should either be loaded into native BQ tables, or optimized via Parquet files for an external BQ table.


**Okay, then why would we ever store data in GCS?**

Cost-Effectiveness: **GCS is significantly cheaper than BigQuery for long-term storage**, especially for data you don't query frequently. This makes it an ideal "data lake" for raw, historical, or rarely accessed data.

Flexibility and Interoperability: **Data in GCS isn't tied to a single Google service**. You can easily share it with other platforms or tools outside of BigQuery, such as Apache Spark, Hadoop, machine learning frameworks (TensorFlow, PyTorch), or custom applications. This provides a flexible "single source of truth" for your organization.

Decoupled Architecture: Storing data in GCS separates your storage from your compute. You can scale your storage needs independently of your query needs. This **allows you to use different compute engines (like BigQuery, Dataproc, or Vertex AI) on the same dataset without moving or duplicating the data**.

## Real-World Example

An enterprise might use GCS and external tables in a data pipeline in a common "data lakehouse" architecture.

**Ingestion**: Raw, messy data from various sources (e.g., IoT devices, application logs, or third-party APIs) is dumped directly into a GCS bucket. This data is often in formats like JSON, CSV, or Avro.

**Transformation (ETL/ELT)**: Use tools like Dataproc (Apache Spark/Hadoop) or Dataflow to perform cleaning, normalization, and aggregation on this raw data. Write the processed, structured data back to GCS in an optimized format like Parquet, with proper partitioning (e.g., by date and event type).

**Consumption**:

Data Exploration & Ad-Hoc Queries: For data scientists or analysts doing exploratory analysis on the newly transformed data, a BigQuery external table is created on top of the partitioned Parquet files in GCS. This allows them to use familiar SQL to query the data without the cost and overhead of loading it into BigQuery. It's a quick, low-cost way to access fresh data.

Dashboarding & Reporting: For critical dashboards and high-performance reporting, the data is loaded from GCS into a native BigQuery table. This is done with a scheduled batch job (e.g., once daily or hourly) using a LOAD DATA statement or a custom pipeline. The native BQ table is then optimized for fast, predictable query performance.

**Real World Example Summary**
In short, it is common to use a combination of both GCS + external tables and native BQ tables. You'd use GCS + external tables for flexibility and cost-effective data exploration on the "hot" or recently processed data. You'd use native BQ tables for performance-critical dashboards, reporting, and applications that require low-latency queries. This hybrid approach gives an enterprise the best of both worlds: the affordability and flexibility of a data lake combined with the speed and power of a data warehouse.

## Rules of Thumb

1. The Volume Rule
Small, ad-hoc, or one-time loads: If you have a small amount of data (megabytes to a few gigabytes) or a one-off task, loading JSON directly into a native BigQuery table is fine. The manual effort of setting up a separate pipeline to convert JSON to Parquet might not be worth the minimal performance gain. BigQuery can handle the parsing overhead for smaller datasets without a significant cost or performance penalty.

Large, recurring data feeds: For large datasets (tens of gigabytes, terabytes, or more) or data you receive on a regular basis, the JSON-to-Parquet transformation is essential. The performance gains from Parquet's columnar format, compression, and partitioning will lead to dramatically lower query costs and faster query execution over time.

2. The Purpose Rule 🎯
Exploratory analysis or data debugging: If you need to quickly look at some data from an API to understand its structure or debug an issue, creating a temporary external table over raw JSON can work. It's a fast way to get a SQL-based view of the data without an ETL process.

Core business intelligence and reporting: For mission-critical dashboards, business intelligence, or data that will be queried by a large number of users, use the JSON-to-Parquet-to-Native-BQ-table pattern. This ensures data is stored in the most performant format possible, providing predictable query times and cost for your most important workflows.

3. The Flexibility vs. Performance Rule ⚖️
Flexibility first: If your data schema is constantly changing and you need to adapt quickly without breaking a pipeline, loading JSON directly into a native BigQuery table using the JSON data type is a valid option. This allows BigQuery to store the raw JSON value, and you can parse it at query time with functions like JSON_VALUE or JSON_QUERY. This trades schema enforcement for flexibility.

Performance first: For a fixed or slowly changing schema, always opt for the JSON-to-Parquet transformation. This process allows you to define and enforce a strict schema, converting nested JSON fields into columns and providing the best possible query performance.

Remember that converting JSON to Parquet is the foundation of a modern data lakehouse architecture, providing a resilient and cost-effective approach for analytical data pipelines.

## Hands-on Exercises with Pyarrow & Parquet

Let's do some examples in a Python notebook (Colab, Hex, Jupyter, etc -- you pick).

The following imports will be assumed throughout the exercises.
```python
# Install the libraries for parquet files
!pip install pyarrow pandas

import pandas as pd
import numpy as np
import pyarrow as pa
import pyarrow.parquet as pq
```

### Exercise 1: Inspect the schema and data types

First, let's create a simple dataframe with Pandas.

```python
# Create a sample Pandas DataFrame
data = {
    'id': range(10),
    'name': ['Gandalf', 'Bilbo', 'Frodo', 'Samwise', 'Aragorn', 'Legolas', 'Gimli', 'Gollum', 'Galadriel', 'Saruman'],
    'age': [25, 30, 35, 40, 45, 50, 55, 60, 65, 70],
    'city': ['New York', 'Los Angeles', 'Chicago', 'Houston', 'Phoenix', 'Philadelphia', 'San Antonio', 'San Diego', 'Dallas', 'San Jose'],
    'is_active': [True, False, True, True, False, True, False, True, True, False],
    'salary': [50000.50, 60000.75, 75000.25, 80000.00, 90000.99, 100000.10, 110000.20, 120000.30, 130000.40, 140000.50]
}

df = pd.DataFrame(data)
print("Sample DataFrame:")
print(df)
```

I'm using Google Colab for the exercises. Next we convert the Pandas dataframe into a PyArrow table.

```py
# Convert the Pandas DataFrame to a PyArrow Table
table = pa.Table.from_pandas(df)

# Write the PyArrow Table to a Parquet file
file_name = 'sample_data.parquet'
pq.write_table(table, file_name)

print(f"\nSuccessfully saved the DataFrame to '{file_name}'")

# Check if the file exists
import os
print(f"File '{file_name}' exists: {os.path.exists(file_name)}")
```

Now we can check the schema and data types of the Pyarrow table in the Parquet file.
```py
# Load the Parquet file using the ParquetFile reader
parquet_file = pq.ParquetFile('sample_data.parquet')

# inspect the
print("Parquet File Schema:")
print(parquet_file.schema)
```

### Exercise 2: Column read performance analysis

One of the biggest benefits of Parquet is columnar storage, which allows you to read only the columns you need. This is a game-changer for performance and cost. We can measure this benefit by reading the parquet file twice -- once reading the whole file, and second reading only specified columns -- and measuring the time required to complete each read. Use the `%timeit` command in Colab to measure and compare the time it takes for each operation.

```py
print("Timing the full file read...")
# Time the full read
%timeit full_df = pd.read_parquet('sample_data.parquet')

print("\nTiming the partial file read (only 'name' and 'age' columns)...")
# Time the partial read
%timeit partial_df = pd.read_parquet('sample_data.parquet', columns=['name', 'age'])
```

It should output something like this:
```
Timing the full file read...
1.91 ms ± 145 µs per loop (mean ± std. dev. of 7 runs, 1000 loops each)

Timing the partial file read (only 'name' and 'age' columns)...
1.75 ms ± 104 µs per loop (mean ± std. dev. of 7 runs, 1000 loops each)
```

We did this on a very small scale, but the column read was still 10% faster. The performance gain is signficantly larger when the table is wider (more columns) and bigger (more rows).

### Exercise 3: Hive Partitioning and Directory Structures

We often partition data to improve query performance and manage data at scale. A partitioned dataset is a collection of Parquet files organized into a directory structure based on one or more column values. In Google Cloud Platform, this is done with GCS buckets. In AWS, we'd use S3. In Azure, it'd be Azure Data Lake Storage Gen2 (ADLS Gen2), which is built on top of Azure Blob Storage. For now, we're just messing around in a Colab notebook file directory, which is suitable for these exercises and the same concept applies.

Create a new DataFrame with a categorical column like department.


```py
data_partition = {
    'employee_id': range(10),
    'department': ['HR', 'IT', 'HR', 'Finance', 'IT', 'HR', 'Finance', 'IT', 'HR', 'Finance'],
    'salary': [50000, 120000, 52000, 75000, 180000, 55000, 85000, 149000, 58000, 92000]
}
df_partition = pd.DataFrame(data_partition)

# Create the dataset
pq.write_to_dataset(
    table=pa.Table.from_pandas(df_partition),
    root_path='departments_partitioned',
    partition_cols=['department']
)

# List the created files and directories
!ls -R departments_partitioned

# Load the entire dataset back into a single DataFrame
df_loaded = pd.read_parquet('departments_partitioned')
print("\nLoaded partitioned DataFrame:")
print(df_loaded)
```

Now in a subsequent notebook cell, run this command:
```bash
!ls -R departments_partitioned
```

The output should look like:
```
departments_partitioned:
'department=Finance'  'department=HR'  'department=IT'

'departments_partitioned/department=Finance':
2fa8c53f47a64de8a71716664efff50a-0.parquet

'departments_partitioned/department=HR':
2fa8c53f47a64de8a71716664efff50a-0.parquet

'departments_partitioned/department=IT':
2fa8c53f47a64de8a71716664efff50a-0.parquet
```

### Exercise 4: Appending Data to Parquet files

In most data pipelines, new data is continuously added (usually either daily batches or streaming) to an existing dataset. Let's simulate this by appending new records to a Parquet dataset. Create a new DataFrame with two new employees for the department dataset from the previous exercise.

```py
# Create the new data
new_employees = {
    'employee_id': [10, 11],
    'department': ['HR', 'IT'],
    'salary': [62000, 155000]
}
new_df = pd.DataFrame(new_employees)

# Convert to PyArrow Table
new_table_to_append = pa.Table.from_pandas(new_df)

# Get the existing dataset
existing_dataset = pq.ParquetDataset('departments_partitioned', use_legacy_dataset=False)

# Write the new data to the existing dataset
# This will write new Parquet files into the correct 'HR' and 'IT' directories
pq.write_to_dataset(
    table=new_table_to_append,
    root_path='departments_partitioned',
    partition_cols=['department'],
    existing_data_behavior='overwrite_or_ignore'
)

# Load the entire dataset again and check for the new employees
df_updated = pd.read_parquet('departments_partitioned')
print("\nUpdated DataFrame with new employees:")
print(df_updated)
```

### Exercise 5: Compression and File Size

Parquet files support various compression algorithms to reduce file size and optimize disk space. Our next task is to save a larger dataset using different compression codecs and compare the results to see the real impact.

Generate a large Pandas DataFrame with a few hundred thousand rows. Include columns with repeating data patterns (e.g., categorical data) as these compress very well. Save this DataFrame to a Parquet file named large_uncompressed.parquet with no compression. Use compression=None. Save the same DataFrame again to a file named large_snappy.parquet using the Snappy compression algorithm. Use compression='snappy'.

Then use the `!ls -lh` shell command to list the file sizes of both files.

```py
# 1. Generate a large DataFrame (e.g., 500,000 rows) with categorical data
num_rows = 500_000
large_df = pd.DataFrame({
    'id': np.arange(num_rows),
    'product_category': np.random.choice(['Electronics', 'Books', 'Home', 'Clothing'], num_rows),
    'sale_amount': np.random.uniform(10, 1000, num_rows),
    'region': np.random.choice(['North', 'South', 'East', 'West'], num_rows)
})

# Convert to PyArrow Table
large_table = pa.Table.from_pandas(large_df)

# 2. Save with no compression
print("Saving uncompressed Parquet file...")
pq.write_table(large_table, 'large_uncompressed.parquet', compression=None)

# 3. Save with Snappy compression
# Snappy is a recommended general-purpose codec for its balance of speed and compression ratio
print("Saving Snappy-compressed Parquet file...")
pq.write_table(large_table, 'large_snappy.parquet', compression='snappy')

# 4. List the file sizes
print("\nComparing file sizes:")
!ls -lh large_uncompressed.parquet large_snappy.parquet

# Explanation: The output of !ls -lh will show that snappy.parquet is significantly smaller than uncompressed.parquet.
# This reduction in size is crucial in big data as it lowers storage costs and reduces I/O time, making data reads faster.
# ```

My output looks like this:
```
Saving uncompressed Parquet file...
Saving Snappy-compressed Parquet file...

Comparing file sizes:
-rw-r--r-- 1 root root 6.5M Sep  1 02:28 large_snappy.parquet
-rw-r--r-- 1 root root 8.5M Sep  1 02:28 large_uncompressed.parquet
```

### Exercise 6: Reading and Filtering with Predicate Pushdown

One of the most powerful features of Parquet is predicate pushdown. This allows the query engine to filter rows at the file level before reading the data into memory, dramatically improving performance for filtered queries.

Load the `large_uncompressed` file from the previous exercise. Use `pd.read_parquet()` to read the data, but this time, pass a filter to the filters parameter to only load records where sale_amount is greater than 500 and the region is North. Print the resulting DataFrame to confirm.

```py
# The large_df from the previous exercise is now saved as 'large_uncompressed.parquet'.
# We will create a partitioned version of this dataset to better demonstrate predicate pushdown.
large_df = pd.read_parquet('large_uncompressed.parquet')

# Partition by 'region'
pq.write_to_dataset(
    pa.Table.from_pandas(large_df),
    root_path='large_partitioned_data',
    partition_cols=['region'],
    compression='snappy'
)

# Load only records where 'region' is 'North' and 'sale_amount' > 500
# This is much more efficient than loading the entire file and then filtering.
north_sales_df = pd.read_parquet(
    'large_partitioned_data',
    filters=[
        ('region', '==', 'North'),
        ('sale_amount', '>', 500)
    ]
)

print("DataFrame loaded using predicate pushdown:")
print(north_sales_df.head())
print(f"\nTotal records loaded: {len(north_sales_df)}")
```

### Exercise 7: From Dataframe to DataLake

This final exercise consolidates your learning by simulating a full end-to-end data engineering task: creating a partitioned, compressed dataset and then performing a filtered query on it. This is similar to exercise 5 but takes it a few steps further.

Create a new DataFrame with columns like timestamp, product_id, and region. Partition this dataset by region and timestamp (for a simple example, a column representing a year or month).

Save the partitioned dataset using Snappy compression.

Then load the dataset using both filters and columns to perform a highly optimized query. For example, "find all product sales for a specific region in a specific month."



```py
# 1. Create a large DataFrame for a sales dataset
num_sales = 500_000
sales_df = pd.DataFrame({
    'sale_id': np.arange(num_sales),
    'product_id': np.random.randint(100, 200, num_sales),
    'region': np.random.choice(['North', 'South', 'East', 'West'], num_sales),
    'sale_amount': np.random.uniform(10, 1000, num_sales),
    'timestamp': pd.to_datetime(pd.Series(np.random.randint(1577836800, 1609459200, num_sales)), unit='s')
})

# Add year and month columns for partitioning
sales_df['year'] = sales_df['timestamp'].dt.year
sales_df['month'] = sales_df['timestamp'].dt.month

# 2. Partition the dataset by 'region' and 'year'
sales_table = pa.Table.from_pandas(sales_df)
pq.write_to_dataset(
    sales_table,
    root_path='sales_data_lake',
    partition_cols=['region', 'year'],
    compression='snappy'
)

# Use !ls to confirm the directory structure
print("Data Lake directory structure:")
!ls -R sales_data_lake

# 3. Perform a highly optimized query using filters and columns
# Find 'product_id' and 'sale_amount' for 'East' region in '2020'
optimized_query_df = pd.read_parquet(
    'sales_data_lake',
    filters=[
        ('region', '==', 'East'),
        ('year', '==', 2020)
    ],
    columns=['product_id', 'sale_amount']
)

print("\nHighly optimized query result:")
print(optimized_query_df.head())
print(f"\nQuery returned {len(optimized_query_df)} records.")

# Explanation: This exercise demonstrates the power of a well-architected data lake.
# The combination of partitioning, compression, and predicate pushdown allows you to perform highly performant queries on massive datasets.
# Parquet's columnar nature and rich metadata make it the ideal format for this kind of analytical workload.
```

My output looks like this, yours might be slightly different:
```
Data Lake directory structure:
sales_data_lake:
'region=East'  'region=North'  'region=South'  'region=West'

'sales_data_lake/region=East':
'year=2020'

'sales_data_lake/region=East/year=2020':
54578c98430a46119350bb621beabe32-0.parquet

'sales_data_lake/region=North':
'year=2020'

'sales_data_lake/region=North/year=2020':
54578c98430a46119350bb621beabe32-0.parquet

'sales_data_lake/region=South':
'year=2020'

'sales_data_lake/region=South/year=2020':
54578c98430a46119350bb621beabe32-0.parquet

'sales_data_lake/region=West':
'year=2020'

'sales_data_lake/region=West/year=2020':
54578c98430a46119350bb621beabe32-0.parquet

Highly optimized query result:
   product_id  sale_amount
0         172   776.702916
1         127   419.763802
2         173   754.848748
3         105   221.194483
4         102   936.093823

Query returned 124923 records.
```

That concludes the Parquet exercise workshop for now, hope this has been helpful to you! Cheers.
