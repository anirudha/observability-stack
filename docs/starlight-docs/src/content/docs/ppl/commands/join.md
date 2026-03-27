---
title: "join"
description: "Combine two datasets together — correlate logs with traces, enrich data from reference indices."
---

import { Tabs, TabItem, Aside } from '@astrojs/starlight/components';

<Aside type="note">
**Stable** since OpenSearch 3.0
</Aside>

The `join` command combines two datasets by matching rows on a condition. The left side is your current pipeline (an index or piped commands); the right side is another index or a subsearch enclosed in square brackets.

Use `join` to correlate logs with traces, enrich spans with service metadata, or combine any two datasets that share a common field.

## Syntax

The `join` command supports two syntax forms: basic and extended.

### Basic syntax

```sql
[joinType] join [left = <leftAlias>] [right = <rightAlias>] (on | where) <joinCriteria> <right-dataset>
```

### Extended syntax

```sql
join [type=<joinType>] [overwrite=<bool>] [max=<n>] (<join-field-list> | [left = <leftAlias>] [right = <rightAlias>] (on | where) <joinCriteria>) <right-dataset>
```

## Arguments

### Basic syntax parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `joinType` | No | The type of join. Valid values: `inner` (default), `left`, `right`, `full`, `semi`, `anti`, `cross`. |
| `left = <leftAlias>` | No | Alias for the left dataset to disambiguate shared field names. Must appear before `right`. |
| `right = <rightAlias>` | No | Alias for the right dataset to disambiguate shared field names. |
| `<joinCriteria>` | Yes | A comparison expression placed after `on` or `where` that specifies how to match rows. |
| `<right-dataset>` | Yes | The right-side dataset. Can be an index name or a subsearch in `[ ... ]`. |

### Extended syntax parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `type` | No | Join type when using extended syntax. Valid values: `inner` (default), `left`, `outer` (alias for `left`), `right`, `full`, `semi`, `anti`, `cross`. |
| `overwrite` | No | When using a field list, whether right-side fields with duplicate names overwrite left-side fields. Default: `true`. |
| `max` | No | Maximum number of right-side matches per left row. Default: `0` (unlimited) when legacy mode is enabled; `1` otherwise. |
| `<join-field-list>` | No | Common field names used to build the join condition automatically. Fields must exist in both datasets. |
| `<joinCriteria>` | Yes | A comparison expression placed after `on` or `where` that specifies how to match rows. |
| `<right-dataset>` | Yes | The right-side dataset. Can be an index name or a subsearch in `[ ... ]`. |

## Join types

| Type | Keyword | Description |
|------|---------|-------------|
| Inner | `inner join` | Returns only rows with a match on both sides. This is the default. |
| Left outer | `left join` | Returns all left rows. Unmatched right fields are `null`. |
| Right outer | `right join` | Returns all right rows. Unmatched left fields are `null`. Requires `plugins.calcite.all_join_types.allowed = true`. |
| Full outer | `full join` | Returns all rows from both sides. Unmatched fields are `null` on the missing side. Requires `plugins.calcite.all_join_types.allowed = true`. |
| Left semi | `left semi join` | Returns left rows that have at least one match on the right. No right-side fields are included. |
| Left anti | `left anti join` | Returns left rows that have **no** match on the right. Useful for finding orphaned records. |
| Cross | `cross join` | Returns the Cartesian product of both sides. Requires `plugins.calcite.all_join_types.allowed = true`. |

## Usage notes

- **Assign aliases** when both sides share field names. Without aliases, ambiguous field names are automatically prefixed with the table name or alias (e.g., `table1.id`, `table2.id`).
- **Keep the right side small.** The right dataset is loaded into memory. Filter or limit the right-side subsearch to keep queries efficient.
- **Right, full, and cross joins** are disabled by default for performance reasons. Enable them by setting `plugins.calcite.all_join_types.allowed` to `true`.
- **Subsearch row limit.** The maximum number of rows from a subsearch is controlled by `plugins.ppl.join.subsearch_maxout` (default: `50000`).
- When using the extended syntax with a **field list**, duplicate field names are deduplicated based on the `overwrite` setting.

## Examples

### Join two indexes

```sql
source = state_country
| inner join left=a right=b ON a.name = b.name occupation
| stats avg(salary) by span(age, 10) as age_span, b.country
```

### Join with a subsearch

```sql
source = state_country as a
| where country = 'USA' OR country = 'England'
| left join ON a.name = b.name [
    source = occupation
    | where salary > 0
    | fields name, country, salary
    | sort salary
    | head 3
  ] as b
| stats avg(salary) by span(age, 10) as age_span, b.country
```

### Join using a field list (extended syntax)

```sql
source = state_country
| where country = 'USA' OR country = 'England'
| join type=left overwrite=true name [
    source = occupation
    | where salary > 0
    | fields name, country, salary
    | sort salary
    | head 3
  ]
| stats avg(salary) by span(age, 10) as age_span, country
```

### Semi join — find events with matches

```sql
source = state_country
| left semi join left=a right=b on a.name = b.name occupation
```

### Anti join — find events without matches

```sql
source = state_country
| left anti join left=a right=b on a.name = b.name occupation
```

## Extended examples

### Correlate logs with trace spans

Join log events with trace span data using `traceId` to see which spans produced each log line:

```sql
source = logs-otel-v1*
| left join left=l right=r on l.traceId = r.traceId [
    source = otel-v1-apm-span-*
    | fields traceId, serviceName, name, durationInNanos
  ]
| fields l.time, l.body, l.severityText, r.serviceName, r.name, r.durationInNanos
| head 50
```

### Enrich spans with service map data

Join raw spans with the service map index to add dependency context:

```sql
source = otel-v1-apm-span-*
| inner join left=span right=svc on span.serviceName = svc.serviceName [
    source = otel-v1-apm-service-map*
    | fields serviceName, destination.domain, destination.resource
    | dedup serviceName
  ]
| fields span.serviceName, span.name, span.durationInNanos, svc.destination.domain
| sort - span.durationInNanos
| head 20
```

## See also

- [lookup](/docs/ppl/commands/lookup/) — simpler enrichment from a reference index
- [subquery](/docs/ppl/commands/) — filter using nested queries
- [append](/docs/ppl/commands/) — stack results vertically instead of joining
- [Command Reference](/docs/ppl/commands/) — all PPL commands
