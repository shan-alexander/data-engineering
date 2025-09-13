# Assert CTEs in SQL

Common Table Expressions (CTEs) refers to a style of writing SQL that contrasts with subqueries. In general, most data engineering, analysis, and Business Intelligence teams will prioritize the use of CTEs over subqueries, for the following reasons:
- **Readability**: asdf
- **Performance**: asdf
- **Troubleshooting**: asdf
- **asdf**: asdf

Most CTEs are written in a form like this psuedocode example:

```sql
with webpages as (
select
  date,
  page_id,
  sum(pageviews) as sum_pageviews
from table1
where date > "2020-02-20"
group by all
)
-- select * from webpages limit 100

, pageclicks as (
select
  a.*,
  sum(b.clicks) as sum_clicks
from webpages wp
left join clicks c on c.page_id = wp.page_id and c.date = wp.date
group by all
)
-- select * from pageclicks limit 100

select
  a.*,
  round(sum(revenue),2) as sum_revenue
from pageclicks pc
left join webpage_revenue wpr on wpr.page_id = pc.page_id and wpr.date = pc.date
group by all
```

Notice the commented-out "select *" checks after each CTE, this is how we would troubleshoot the output of each CTE to verify the result values are correct, formatted correctly, etc.

However, I've never seen someone write CTE queries with ASSERTs.

In this writeup, **I propose that all CTE queries SHOULD use one or more ASSERTS after each CTE**. Therefore, the psuedocode example should look like this instead:

```sql
with webpages as (
select
  date,
  page_id,
  sum(pageviews) as sum_pageviews
from table1
where date > "2020-02-20"
group by all
)
-- select * from webpages limit 100

assert(
  select count(*) > 1000 AND COUNTIF(page_id IS NULL) = 0 AND min(sum_pageviews) > 0
  FROM webpages
) as 'Webpages have at least 1 pageview and no null page_ids. Should return more than 1000 rows.'

, pageclicks as (
select
a.*,
sum(b.clicks) as sum_clicks
from webpages wp
left join clicks c on c.page_id = wp.page_id and c.date = wp.date
group by all
)
-- select * from pageclicks limit 100

assert(
  select count(*) > 1000 AND
  COUNTIF(page_id IS NULL) = 0 AND
  min(sum_pageviews) > 0 AND
  min(sum_clicks) >= 0
  FROM pageclicks
) as 'Webpages should have 0 or more clicks and previous asserts met.'

, final_output as (
select
a.*.
round(sum(revenue),2) as sum_revenue
from pageclicks pc
left join webpage_revenue wpr on wpr.page_id = pc.page_id and wpr.date = pc.date
group by all
)
-- select * from final_output limit 100

assert(
  select count(*) > 1000 AND
  COUNTIF(page_id IS NULL) = 0 AND
  min(sum_clicks) >= 0
  min(sum_revenue) >= 0 AND
  max(sum_revenue) < 1000000 AND
  COUNTIF(sum_revenue is null) = 0 AND
  COUNT(*) = COUNT(DISTINCT page_id) --no dupes
  FROM final_output
) as 'Revenue column values should not be null, should be a float of 0 or more.'
```

The difference in the example above is that we apply ASSERT conditions to the output of each CTE, and we give each assert a particular message to throw in the error message if the assert does not pass.

Why not put all the ASSERTs in a separate script, and run the assert script in a workflow? Because then, the script can only test the final output, and does not test each CTE. Therefore, troubleshooting will be more difficult. If each CTE has its own ASSERTs and each assert has a unique error message, we will know exactly which CTE is causing the fail.

This is a powerful concept!

I've written thousands of complex SQL queries with long chains of CTEs, and never wrote with ASSERTs after each CTE. As of 2025, all my CTEs receive their own ASSERTs, and I believe it's the most technically-excellent approach to ensuring data quality, maintainability, ease of troubleshooting, etc. Of course we still want DBT/Dataform assertions in the config block, or an assert script in our workflows, but these only assert the final output and not the output of each CTE. If we write asserts for each CTE, our ability to troubleshoot is improved, and the data asset is increased in value (high trust).

## Additional thoughts on data quality, performance, & technical excellence

I often see other SQL authors using row_number() to isolate the most recent row of an entity. For example, lets say we have a data of a content management system like Wordpress, which creates a new version of the webpage each time we update it. All our previous versions of the webpage are stored in a historical log table. Similar to the SQL examples above, we want to get pageviews, clicks, and revenue (a common web analytics ask) for each webpage, with daily totals of those pageviews, clicks, and revenue.

Many SQL authors will write in this approach:

```sql
WITH raw_versions AS (
  SELECT
    version_id,
    webpage_id,
    version_number,
    created_date,
    name
  FROM `your-project.your_dataset.webpage_versions`
),
ranked_versions AS (
  SELECT
    *,
    ROW_NUMBER() OVER (
      PARTITION BY webpage_id
      ORDER BY created_date DESC, version_number DESC
    ) AS rn
  FROM raw_versions
),
current_webpages AS (
  SELECT EXCEPT(rn)
  FROM ranked_versions
  WHERE rn = 1
)
select * from current_webpages
```

The example above is the worst approach, and unfortunately the most common (ime). I've cleaned up dozens of queries, making them execute 50% to 60% faster, by converting the window functions to a max/min function. The window functions are the most expensive computation in SQL (sorting/ordering) whereas max/min functions rely on hashing (significantly more performant).

I would rewrite the query like this:
```sql
WITH max_versions AS (
  SELECT
    webpage_id,
    max(version_number) as max_version_number,
  FROM `your-project.your_dataset.webpage_versions`
),
current_versions AS (
  SELECT
    webpage_id,
    version_id
    version_number,
    created_date,
    name
  FROM max_versions mv
  inner join `your-project.your_dataset.webpage_versions` wv
    on wv.webpage_id = mv.webpage_id
    and wv.version_number = mv.max_version_number
)
select * from current_versions
```

In the improved example, I use a max() function for getting the most recent webpage version number. If the version_id field is incremental, I could apply the max() function to it, but some ID fields are UUIDs hence me relying on the version_number column here. Not only did I reduce `bytes shuffled` by reducing the number of CTE data held in memory by the SQL engine, I drastically improved performance by eliminating a sort (the row_number window function).

However, I want to acknowledge that once in a blue moon, I'll encounter data in which the max/min approach cannot work. Sometimes, there are "ties" in which two or more rows will return after using the max() function. The row_number() approach will order those tying-rows and only one of the ties will get a row_num value of 1, so it seems we're forced to use expensive sorting in this case.

For such situations, I recommend to use `QUALIFY` instead of the older approach of two CTEs, one for row_number() and the 2nd CTE for `SELECT EXCEPT(rn) ... WHERE rn = 1`.

Using the same code example, here's how we use `QUALIFY`:

```sql
WITH current_webpages AS (
  SELECT
    version_id,
    webpage_id,
    version_number,
    created_date,
    name
  FROM raw_versions
  QUALIFY ROW_NUMBER() OVER (
    PARTITION BY webpage_id
    ORDER BY created_date DESC, version_number DESC
  ) = 1
)

SELECT * FROM current_webpages
```

The `QUALIFY` function allows us to condense the entire window function and  `SELECT EXCEPT(rn) ... WHERE rn = 1` logic into 1 CTE for readability and potentially less bytes shuffled than chaining CTEs. A nice assert to follow qualify filters is `COUNT(*) = COUNT(DISTINCT webpage_id)` which ensures one row per that asserted ID.

Virtually every team and SQL author can improve legacy code by applying the principles mentioned in this document:

- Replace window functions with hash aggregations where possible for reducing both compute and bytes shuffled.
- Use `QUALIFY` instead of a 2nd CTE to isolate row_num=1, when a min/max function cannot be used.
- Use asserts after each CTE for improved data quality & troubleshooting.
