# Queries

Starting points in HogQL, run through the PostHog connector's SQL query capability. Placeholders in
`{braces}` come from the config and the run's window. Check results against the PostHog UI the first
time; event and property names vary between projects.

## Excluding internal users

Add to every query on `events`:

```sql
AND NOT match(coalesce(person.properties.email, ''), '@({domain_1}|{domain_2})$')
AND distinct_id NOT IN ({internal_distinct_ids})
-- with a cohort: AND person_id NOT IN COHORT {cohort_id}
```

## Errors: one row per issue, window vs window before vs baseline

```sql
SELECT
  properties.$exception_issue_id AS issue_id,
  any(properties.$exception_message) AS message,
  any(properties.$current_url) AS url,
  countIf(timestamp >= {start} AND timestamp < {end}) AS in_window,
  countIf(timestamp >= {start} - INTERVAL {window_minutes} MINUTE AND timestamp < {start}) AS previous_window,
  countIf(timestamp < {start}) / ({baseline_days} * 1440 / {window_minutes}) AS baseline,
  uniqIf(person_id, timestamp >= {start} AND timestamp < {end}) AS users,
  groupUniqArrayIf(10)(properties.{group_property}, timestamp >= {start} AND timestamp < {end}) AS groups
FROM events
WHERE event = '$exception'
  AND timestamp >= {start} - INTERVAL {baseline_days} DAY
  AND timestamp < {end}
  -- internal-user exclusions
GROUP BY issue_id
HAVING in_window > 0
ORDER BY in_window DESC
```

- **New:** the issue's first-seen time (from error tracking's issue list) falls in `[start, end)`.
- **Spiking:** `in_window >= spike_multiplier * greatest(baseline, 1)` and `in_window >= spike_min_count`,
  and `previous_window` doesn't pass the same test.
- Drop rows whose `message` matches `ignore_errors`, and issues in `known_issues` unless spiking.

## Activation: funnel

Use a funnel query (not HogQL) over `funnel_steps`, person-level, conversion window
`conversion_window_days`, broken down by `group_property`. Run it twice: the current `window_days` and
the one before. Report each step's conversion and the change in points.

## Activation: time to value

```sql
SELECT
  quantile(0.5)(dateDiff('minute', first_from, first_to)) AS median_minutes,
  count() AS persons
FROM (
  SELECT
    person_id,
    minIf(timestamp, event = '{from}') AS first_from,
    minIf(timestamp, event = '{to}') AS first_to
  FROM events
  WHERE event IN ('{from}', '{to}')
    AND timestamp >= {window_start} AND timestamp < {window_end}
    -- internal-user exclusions
  GROUP BY person_id
)
WHERE first_from > toDateTime(0) AND first_to >= first_from
```

## Activation: what drives a friction event

```sql
SELECT properties.{property} AS value, count() AS events, uniq(person_id) AS persons
FROM events
WHERE event = '{friction_event}'
  AND timestamp >= {window_start} AND timestamp < {window_end}
  -- internal-user exclusions
GROUP BY value
ORDER BY persons DESC
LIMIT 10
```

Run it for the event's two or three most telling properties (a field name, a reason, a page).
