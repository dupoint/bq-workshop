<h2>General resources</h2>
<p>Below are some links to resources frequently used throughout this course.</p>
<ul>
<li>
GA4 Merchandise store (Demo GA4 account): 
<a target="_blank" href="https://analytics.google.com/analytics/web/?utm_source=demoaccount&utm_medium=demoaccount&utm_campaign=demoaccount#/p213025502/reports/intelligenthome">Link</a>
</li>
<li>
GA4 Sample BigQuery dataset: <a target="_blank" href="https://console.cloud.google.com/bigquery?p=bigquery-public-data&d=ga4_obfuscated_sample_ecommerce&t=events_20210131&page=table" >
  ga4_obfuscated_sample_ecommerce
</a>
</li>
<li>
BigQuery quick intro lab (Cloud Skills Boost): <a target="_blank" href="https://www.cloudskillsboost.google/focuses/1145?catalog_rank=%7B%22rank%22%3A10%2C%22num_filters%22%3A0%2C%22has_search%22%3Atrue%7D&parent=catalog&search_id=33376315" >
  Link
</a>

  
</ul>

<h2>BigQuery Workshop</h2>
<h3>SQL Queries</h3>
<strong>Query 1: Get count of total GA4 events fired </strong>


<pre>
/* Count of total events */
  SELECT
      COUNT(event_name) AS event_count
    FROM
      `think-bigquery.analytics_338032405.events_20250622`

</pre>

<strong>Query 1.5: Get count of total GA4 events by event name </strong>


<pre>
/* Count of total events */
  SELECT
    event_name,
    COUNT(event_name) AS event_count
    FROM      `think-bigquery.analytics_338032405.events_20250622`
GROUP BY event_name
ORDER BY event_count DESC
</pre>
<strong>Query 2: Get count of GA4 key events fired </strong>


<pre>
/* Count of contact-us events */
  SELECT
      event_name,
      SUM(
        CASE
          WHEN event_name = 'contactUs' THEN 1
          ELSE 0
        END
      ) AS contactUs
    FROM
      `think-bigquery.analytics_338032405.events_20250622`
    GROUP BY event_name
</pre>
<strong>Gemini Prompt 1.1: Flatten event parameter values </strong>
<pre>
Extract event_date, 
event_timestamp, 
event_name, 
event_params_* as a subtable t1 
from table
where event_params_* is flattened from the nested field event_params

NOTE: duplicated values for event_date, event_timestamp, event_name are allowed in the rows for t1, also create columns for each event_params nested value 
for e.g. ep_key, ep_int, ep_float, ep_double etc

</pre>
<strong>Query output from prompt 1.1: Flatten event parameter values </strong>
<pre>
SELECT
    events.event_date,
    events.event_timestamp,
    events.event_name,
    event_params.key AS ep_key,
    event_params.value.int_value AS ep_int,
    event_params.value.float_value AS ep_float,
    event_params.value.double_value AS ep_double,
    event_params.value.string_value AS ep_string
  FROM
    `think-bigquery.analytics_338032405.events_20210106` AS events,
    UNNEST(events.event_params) AS event_params;
</pre>


<strong>Gemini Prompt 1.2: Count total sessions using CTE </strong>
<pre>
Use as cte (
SELECT
    events.event_date,
    events.event_timestamp,
    events.event_name,
    event_params.key AS ep_key,
    event_params.value.int_value AS ep_int,
    event_params.value.float_value AS ep_float,
    event_params.value.double_value AS ep_double,
    event_params.value.string_value AS ep_string
  FROM
    `think-bigquery.analytics_338032405.events_20210106` AS events,
    UNNEST(events.event_params) AS event_params;
) 
Get unique count of (user_pseudo_id concatanated with ep_int, WHEN ep_key =
 'ga_session_id') FROM cte
NOTE: Leave the cte query unchanged 

</pre>
<strong>Query output from prompt 1.2: Count total sessions using CTE </strong>
<pre>
WITH cte AS (
    SELECT
      events.event_date,
      events.event_timestamp,
      events.event_name,
      event_params.key AS ep_key,
      event_params.value.int_value AS ep_int,
      event_params.value.float_value AS ep_float,
      event_params.value.double_value AS ep_double,
      event_params.value.string_value AS ep_string,
      events.user_pseudo_id
    FROM
      `think-bigquery.analytics_338032405.events_20210106` AS events,
      UNNEST(events.event_params) AS event_params
  )
SELECT
    count(DISTINCT concat(cte.user_pseudo_id, CAST(ep_int as STRING)))
  FROM
    `cte`
  WHERE cte.ep_key = 'ga_session_id';
</pre>



<strong>Query 3: Get count of total users and sessions  </strong>


<pre>
/* Total users and sessions */
  SELECT
    COUNT(DISTINCT user_pseudo_id) AS total_users,
    COUNT(DISTINCT session_id) AS sessions
  FROM
    (
      SELECT
        user_pseudo_id,
        CONCAT(
          user_pseudo_id,
          (
            SELECT
              value.int_value
            FROM
              UNNEST (event_params)
            WHERE
              key = 'ga_session_id'
          )
        ) AS session_id
      FROM
        `think-bigquery.analytics_338032405.events*`
      GROUP BY
        user_pseudo_id,
        session_id
    )
</pre>

<h3>Gemini Prompts</h3>
<strong>System prompt</strong>
<pre>
  Constraints:
        - GA4 tables are daily: `{PROJECT_ID}.{BIGQUERY_DATASET_ID}.events_*` (wildcard).
        - If you query events_* (wildcard), you MUST add an `_TABLE_SUFFIX BETWEEN 'YYYYMMDD' AND 'YYYYMMDD'`
          filter in the SAME SELECT that scans events_*.
        - If you query a single daily table like `events_YYYYMMDD`, do NOT use `_TABLE_SUFFIX`.
        - Always reference the table as ONE fully backticked identifier:
          use ```{PROJECT_ID}.{BIGQUERY_DATASET_ID}.events_*``` inside a single pair of backticks,
          never like ```{PROJECT_ID}```.{BIGQUERY_DATASET_ID}.events_*.
        - `event_date` is STRING 'YYYYMMDD'; use `PARSE_DATE('%Y%m%d', event_date)` for date math.
        - Sessions (precise) = DISTINCT CONCAT(user_pseudo_id,'-', ga_session_id via event_params).
        - For large windows, it's OK to approximate sessions as COUNT(*) of `session_start`.
        - When you need multiple keys from event_params, DO NOT UNNEST(event_params) twice.
          Use scalar subqueries like (SELECT ep.value.X FROM UNNEST(event_params) ep WHERE ep.key='...').
        - Keep queries within bytes limits and clamp any range to: {window_hint}.
        - For “sessions over a period broken down by <dimension>, grouped by month/day”, compute
          COUNT(*) of `session_start` grouped by (FORMAT_DATE('%Y-%m', PARSE_DATE('%Y%m%d', event_date)) or PARSE_DATE('%Y%m%d', event_date), <dimension>),
          limited to a top-N of allowed dimensions: {", ".join(sorted(DIM_MAP.keys()))}.
        - If custom event names are specified like "for events (name1, name2, name3)", then always use the eventnames exactly as listed without changes
        - Never read dimensions like `device_category`, `source`, `medium`, `campaign`, `country`, `region`, `city`,
          `language`, `browser`, `operating_system` from `event_params`. Use the GA4 export columns:
          device.*, traffic_source.*, geo.*, etc.
</pre>

<strong>Prompt 1: </strong>
<pre>
<i>
 Get count of all events broken down by event_name and week (from event_date) from table:
 `think-bigquery.analytics_338032405.events*`
 order events by event count descending order and week ascending order. 
Note that event_date is a string value, so convert this string to a date format with PARSE_DATE before extracting the week
  </i>
</pre>

<pre>
WITH cte_flat AS
(
  SELECT
    event_date,
    event_timestamp,
    event_name,
    user_pseudo_id,
    event_param.key AS pkey,
    event_param.value AS pvalue
  FROM
    `think-bigquery.analytics_338032405.events*`,
    UNNEST(event_params) AS event_param
  WHERE event_param.key IN(
    'ga_session_id', 'session_engaged', 'engagement_time_msec', 'ga_session_number'
  )
)
SELECT COUNT(DISTINCT user_pseudo_id) AS total_users,
COUNT(
    DISTINCT(
            CASE WHEN pkey="ga_session_id"
            THEN CONCAT(CAST(pvalue.int_value AS STRING),CAST(user_pseudo_id AS STRING))
            ELSE NULL END 
            )

      ) AS sessions

FROM cte_flat 
</pre>
<strong>Prompt 2: </strong>
/* Conversions data query */
<pre>
get total sessions, event counts for events (contactUs, courseGuide, eventRegistration) grouped by month, for the past 4 months, broken down by source_medium dimension
</pre>
<strong>Visual Prompt 1: </strong>

<pre>
Area chart, plotting sessions (y axis) segmented by source dim over months (x axis)
</pre>

<strong>Visual Prompt 2: </strong>

<pre>
Area chart, plotting sessions (y axis) segmented by source dim over months (x axis), remove gaps in the data so the graphs are smooth, show dates in "MM-YYYY" format
</pre>
<strong>Advanced analysis prompt: </strong>
<pre>
what source and medium combination generated the highest contactus events, and in which month?</pre>
