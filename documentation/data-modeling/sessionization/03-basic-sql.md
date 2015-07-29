---
layout: page
header: title
group: documentation
subgroup: data modeling
breadcrumb: sessionization
subbreadcrumb: basic server-side sessionization in sql
weight: 3
title:
description: Basic sessionization in SQL.
permalink: /documentation/data-modeling/sessionization/basic-sessionization-in-sql/
---

How to aggregate events into sessions.

{% highlight sql %}

WITH step_1 AS (
  SELECT
    id,
    tstamp,
    LAG(tstamp, 1) OVER (PARTITION BY id ORDER BY tstamp) AS previous_tstamp
  FROM events
), step_2 AS (
  SELECT
    id,
    tstamp,
    previous_tstamp,
    CASE WHEN EXTRACT(EPOCH FROM (tstamp - previous_tstamp)) < 60*30 THEN 0 ELSE 1 END AS new_session
  FROM step_1
), step_3 AS (
  SELECT
    id,
    tstamp,
    previous_tstamp,
    new_session,
    SUM(new_session) OVER (PARTITION BY id ORDER BY tstamp ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS session_idx
  FROM step_2
)

SELECT
  id,
  session_idx,
  MIN(tstamp) AS min_tstamp,
  COUNT(*) AS event_count
FROM step_3
GROUP BY id, session_idx
{% endhighlight %}

<img src="/assets/img/documentation/sessionization/basic.png" width="282px">

{% highlight sql %}
SELECT
  id,
  tstamp,
  LAG(tstamp, 1) OVER (PARTITION BY id ORDER BY tstamp) AS previous_tstamp
FROM events
{% endhighlight %}

<img src="/assets/img/documentation/sessionization/basic-previous.png" width="466px">

{% highlight sql %}
SELECT
  id,
  tstamp,
  previous_tstamp,
  CASE WHEN EXTRACT(EPOCH FROM (tstamp - previous_tstamp)) < 60*30 THEN 0 ELSE 1 END AS new_session
FROM (...)
{% endhighlight %}

<img src="/assets/img/documentation/sessionization/basic-delta.png" width="600px">

{% highlight sql %}
SELECT
  id,
  tstamp,
  previous_tstamp,
  new_session,
  SUM(new_session) OVER (PARTITION BY id ORDER BY tstamp ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS session_idx
FROM (...)
{% endhighlight %}

<img src="/assets/img/documentation/sessionization/basic-index.png" width="737px">

{% highlight sql %}
SELECT
  id,
  session_idx,
  MIN(tstamp) AS min_tstamp,
  COUNT(*) AS event_count
FROM (...)
GROUP BY id, session_idx
{% endhighlight %}

<img src="/assets/img/documentation/sessionization/basic-aggregated.png" width="601px">

## What columns to use?

The previous example

What ID to use?

That depends on the events you are interested in. Examples: domain_userid, user_id, apple_idfa