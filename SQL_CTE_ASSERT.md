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
  COUNTIF(sum_revenue is null) = 0
  FROM final_output
) as 'Revenue column values should not be null, should be a float of 0 or more.'
```

The difference in the example above is that we apply ASSERT conditions to the output of each CTE, and we give each assert a particular message to throw in the error message if the assert does not pass.

Why not put all the ASSERTs in a separate script, and run the assert script in a workflow? Because then, the script can only test the final output, and does not test each CTE. Therefore, troubleshooting will be more difficult. If each CTE has its own ASSERTs and each assert has a unique error message, we will know exactly which CTE is causing the fail.

This is a powerful concept!

I've written thousands of complex SQL queries with long chains of CTEs, and never wrote with ASSERTs after each CTE. As of 2025, all my CTEs receive their own ASSERTs, and I believe it's the most technically-excellent approach to ensuring data quality, maintainability, ease of troubleshooting, etc. Of course we still want DBT/Dataform assertions in the config block, or an assert script in our workflows, but these only assert the final output and not the output of each CTE. If we write asserts for each CTE, our ability to troubleshoot is improved, and the data asset is increased in value (high trust).
