
## Updated nested object schema
```sql
gcloud config set project $project
bq show --schema ${table} > tmp.json

! Modify tmp.json !

bq update ${table} tmp.json
```

## Get queries by bytes scanned 
```sql
DECLARE timezone STRING DEFAULT "America/Los_Angeles";
DECLARE gb_divisor INT64 DEFAULT 1024*1024*1024;
DECLARE tb_divisor INT64 DEFAULT gb_divisor*1024;
DECLARE cost_per_tb_in_dollar INT64 DEFAULT 5;
DECLARE cost_factor FLOAT64 DEFAULT cost_per_tb_in_dollar / tb_divisor;

WITH cost_per_query as (
  SELECT
    p.job_id,
    DATE(p.creation_time, timezone) creation_date,
    FORMAT_TIMESTAMP("%F %H:%I:%S", p.creation_time, timezone) as query_time,  
    ROUND(p.total_bytes_processed / gb_divisor,2) as bytes_processed_in_gb,
    IF(p.cache_hit != true, ROUND(p.total_bytes_processed * cost_factor,4), 0) as cost_in_dollar,
    p.project_id,
    p.user_email,
    j.statement_type,
    j.query
  FROM 
    `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT as p
    JOIN `region-us`.INFORMATION_SCHEMA.JOBS as j
      ON p.job_id = j.job_id
  WHERE
    DATE(p.creation_time) BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY) and CURRENT_DATE()
)

SELECT
  query,
  SUM(cost_in_dollar) as total_cost,
  COUNT(*) as count,
  SAFE_DIVIDE(SUM(cost_in_dollar), COUNT(*))
FROM cost_per_query
GROUP BY query
HAVING SUM(cost_in_dollar) > 50
ORDER BY SUM(cost_in_dollar) DESC
```