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
      `cotton-on-e41b2.analytics_195776711.events_20250622`
</pre>
<strong>Query 1.5: Get count of total GA4 events by event name </strong>


<pre>
/* Count of total events */
  SELECT
    event_name,
    COUNT(event_name) AS event_count
    FROM      `cotton-on-e41b2.analytics_195776711.events_20250622`
GROUP BY event_name
ORDER BY event_count DESC
</pre>
<strong>Query 2: Get count of GA4 purchase events fired </strong>


<pre>
/* Count of purchase events */
  SELECT
      event_name,
      SUM(
        CASE
          WHEN event_name = 'purchase' THEN 1
          ELSE 0
        END
      ) AS purchases
    FROM
      `cotton-on-e41b2.analytics_195776711.events_20250622`
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
    `cotton-on-e41b2.analytics_195776711.events_20210106` AS events,
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
    `cotton-on-e41b2.analytics_195776711.events_20210106` AS events,
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
      `cotton-on-e41b2.analytics_195776711.events_20210106` AS events,
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
        `cotton-on-e41b2.analytics_195776711.events*`
      GROUP BY
        user_pseudo_id,
        session_id
    )
</pre>

<h3>Gemini Prompts</h3>
<strong>Prompt 1: </strong>
<pre>
<i>
 Get count of all events broken down by event_name and week (from event_date) from table:
 `cotton-on-e41b2.analytics_195776711.events*`
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
    `cotton-on-e41b2.analytics_195776711.events*`,
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
/* Ecommmerce data query */
<pre>
get top selling items by total revenue broken down by weeks sorted by revenue descending
</pre>pre>

