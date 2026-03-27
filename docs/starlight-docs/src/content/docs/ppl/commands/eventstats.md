---
title: "eventstats"
description: "Add aggregation statistics as new fields to every event — enrich each row with group-level context."
---

import { Tabs, TabItem, Aside } from '@astrojs/starlight/components';

<Aside type="caution">
**Experimental** since OpenSearch 3.1 — syntax may change based on community feedback.
</Aside>

The `eventstats` command calculates summary statistics and appends them as new fields to **every** event in your results. Unlike `stats`, which collapses rows into an aggregation table, `eventstats` preserves every original event and adds the computed values alongside.

This makes `eventstats` ideal for comparing individual events against group-level context -- flagging outliers, calculating deviation from the norm, or adding percentile baselines to each row.

## Syntax

```sql
eventstats [bucket_nullable=<bool>] <function>... [by <by-clause>]
```

## Arguments

| Parameter | Required | Description |
|-----------|----------|-------------|
| `<function>` | Yes | One or more aggregation functions (e.g., `avg(field)`, `count()`, `max(field)`). Each produces a new field in every row. |
| `bucket_nullable` | No | Whether `null` values form their own group in `by` aggregations. Default is controlled by `plugins.ppl.syntax.legacy.preferred`. |
| `<by-clause>` | No | Group results by one or more fields or expressions. Syntax: `by [span-expression,] [field,]...`. Without a `by` clause, statistics are computed across all events. |
| `span(<field>, <interval>)` | No | Split a numeric or time field into buckets. Example: `span(age, 10)` creates 10-unit buckets; `span(time, 1h)` creates hourly buckets. |

### Time units for span expressions

`ms` (milliseconds), `s` (seconds), `m` (minutes), `h` (hours), `d` (days), `w` (weeks), `M` (months), `q` (quarters), `y` (years).

## Supported aggregation functions

`COUNT`, `SUM`, `AVG`, `MAX`, `MIN`, `VAR_SAMP`, `VAR_POP`, `STDDEV_SAMP`, `STDDEV_POP`, `DISTINCT_COUNT` / `DC`, `EARLIEST`, `LATEST`.

## Usage notes

- **Use `eventstats` when you need both the raw event and the aggregate.** If you only need the aggregation table, use `stats` instead -- it is faster.
- **Combine with `eval` and `where`** to calculate deviations or filter outliers. For example, compute `avg(latency)` per service with `eventstats`, then `eval deviation = latency - avg_latency` and `where deviation > threshold`.
- **Span expressions** let you bucket time or numeric fields, which is useful for comparing events within time windows.
- **`bucket_nullable=false`** excludes rows with `null` group-by values from aggregation (their aggregated field is also `null`). Use `bucket_nullable=true` to treat `null` as a valid group.

## Examples

### Average, sum, and count by group

Calculate aggregate statistics per gender and add them to every row:

```sql
source = accounts
| fields account_number, gender, age
| eventstats avg(age), sum(age), count() by gender
| sort account_number
```

Every row retains its original fields, plus new `avg(age)`, `sum(age)`, and `count()` columns with the group-level values.

### Count by span and group

Count events within 5-year age buckets, grouped by gender:

```sql
source = accounts
| fields account_number, gender, age
| eventstats count() as cnt by span(age, 5) as age_span, gender
| sort account_number
```

### Filter after enrichment

Add the service-level average, then keep only events that deviate significantly:

```sql
source = accounts
| eventstats avg(age) as avg_age by gender
| eval deviation = age - avg_age
| where abs(deviation) > 5
| fields account_number, gender, age, avg_age, deviation
```

### Null bucket handling

Exclude `null` group-by values from aggregation:

```sql
source = accounts
| eventstats bucket_nullable=false count() as cnt by employer
| fields account_number, firstname, employer, cnt
| sort account_number
```

Rows where `employer` is `null` receive `null` for `cnt`.

## Extended examples

### Add service average latency to each span

Compute per-service average latency and attach it to every span, then identify outliers:

```sql
source = otel-v1-apm-span-*
| eventstats avg(durationInNanos) as avg_duration by serviceName
| eval deviation = durationInNanos - avg_duration
| where deviation > avg_duration * 2
| fields serviceName, name, durationInNanos, avg_duration, deviation
| sort - deviation
| head 20
```

### Flag high-severity log spikes per service

Count logs per service and severity, then flag services with unusually high error counts:

```sql
source = logs-otel-v1*
| eventstats count() as svc_error_count by `resource.attributes.service.name`, severityText
| where severityText = 'ERROR'
| eventstats avg(svc_error_count) as avg_errors
| where svc_error_count > avg_errors * 3
| fields `resource.attributes.service.name`, severityText, svc_error_count, avg_errors
| dedup `resource.attributes.service.name`
```

## See also

- [stats](/docs/ppl/commands/) — aggregate and collapse rows
- [streamstats](/docs/ppl/commands/streamstats/) — cumulative and rolling window statistics
- [trendline](/docs/ppl/commands/trendline/) — moving averages
- [Command Reference](/docs/ppl/commands/) — all PPL commands
