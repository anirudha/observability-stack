---
title: "sort"
description: "Sort search results by one or more fields in ascending or descending order."
---

## Description

The `sort` command orders search results by one or more fields. It supports ascending and descending order, multiple sort keys, null value ordering, and type-specific sorting functions. Use it to find top-N results, order events chronologically, or rank aggregated data.

PPL supports two notation styles for specifying sort direction -- prefix notation (`+ field` / `- field`) and suffix notation (`field asc` / `field desc`). Both produce identical results; choose whichever reads more clearly for your query. You must use one notation style consistently within a single `sort` command.

---

## Syntax

### Prefix notation

```sql
sort [<count>] [+|-] <field> [, [+|-] <field>]...
```

### Suffix notation

```sql
sort [<count>] <field> [asc|desc|a|d] [, <field> [asc|desc|a|d]]...
```

---

## Arguments

| Parameter | Required | Description |
|-----------|----------|-------------|
| `<field>` | Yes | The field to sort by. Multiple fields are comma-separated; earlier fields take priority. Use `auto(field)`, `str(field)`, `ip(field)`, or `num(field)` to control how values are interpreted. |
| `+` / `-` | No | **Prefix notation only.** `+` for ascending (default), `-` for descending. |
| `asc` / `desc` | No | **Suffix notation only.** `asc` (or `a`) for ascending (default), `desc` (or `d`) for descending. |
| `<count>` | No | Maximum number of results to return. `0` or omitted returns all results. Equivalent to piping through `head`. |

---

## Usage notes

- **Default order is ascending**: If you omit the direction indicator, results are sorted in ascending order (smallest/earliest first).

- **Null and missing values**: Null values sort first in ascending order and last in descending order. This is important when sorting fields that may not exist on every document.

- **Type-specific sort functions**: Control how field values are compared:
  - `auto(field)` -- automatic type detection (default behavior).
  - `str(field)` -- sort as strings (lexicographic). Useful for sorting numeric fields as text (e.g. `str(account_number)` makes `"13"` come before `"6"`).
  - `num(field)` -- sort as numbers.
  - `ip(field)` -- sort as IP addresses.

- **Count parameter for top-N queries**: `sort 10 - duration` returns only the 10 events with the highest duration. This is more efficient than `sort - duration | head 10` because it can optimize internally.

- **Multi-field sorting**: Fields are evaluated left to right. If two records tie on the first field, the second field breaks the tie, and so on.

- **Performance**: Sorting large result sets is memory-intensive because all matching documents must be held and compared. For large datasets, combine `sort` with `stats` aggregation or use `head` to limit results. Sorting after `stats` (which typically produces fewer rows) is much cheaper than sorting raw events.

- **Do not mix notations**: Use either prefix or suffix notation within a single `sort` command -- mixing `- age, name desc` in one command is not supported.

---

## Basic examples

### Sort ascending (default)

```sql
source = accounts
| sort age
| fields account_number, age
```

### Sort descending with prefix notation

```sql
source = accounts
| sort - age
| fields account_number, age
```

### Multi-field sort

Sort by gender ascending, then by age descending:

```sql
source = accounts
| sort + gender, - age
| fields account_number, gender, age
```

This is equivalent in suffix notation:

```sql
source = accounts
| sort gender asc, age desc
| fields account_number, gender, age
```

### Limit results with count

Return only the two youngest accounts:

```sql
source = accounts
| sort 2 age
| fields account_number, age
```

### Lexicographic sort with `str()`

Sort numeric field as strings (lexicographic order):

```sql
source = accounts
| sort str(account_number)
| fields account_number
```

---

## Extended examples

### OTel: Most recent error logs

Retrieve the 20 most recent error logs across all services, sorted by timestamp descending.

```sql
| where severityText = 'ERROR'
| sort - time
| head 20
| fields time, `resource.attributes.service.name`, body
```

<a href="https://observability.playground.opensearch.org/w/19jD-R/app/explore/logs/#/?_g=(filters:!(),refreshInterval:(pause:!t,value:0),time:(from:now-15m,to:now))&_q=(dataset:(id:d1f424b0-2655-11f1-8baa-d5b726b04d73,timeFieldName:time,title:'logs-otel-v1*',type:INDEX_PATTERN),language:PPL,query:'%7C%20where%20severityText%20%3D%20%27ERROR%27%20%7C%20sort%20-%20time%20%7C%20head%2020%20%7C%20fields%20time%2C%20%60resource.attributes.service.name%60%2C%20body')&_a=(legacy:(columns:!(body,severityText,resource.attributes.service.name),interval:auto,isDirty:!f,sort:!()),tab:(logs:(),patterns:(usingRegexPatterns:!f)),ui:(activeTabId:logs,showHistogram:!t))" target="_blank" rel="noopener">Try in playground &rarr;</a>

### OTel: Services with the most log volume

Aggregate log counts by service, then sort to find the noisiest services.

```sql
| stats count() as log_count by `resource.attributes.service.name`
| sort - log_count
```

<a href="https://observability.playground.opensearch.org/w/19jD-R/app/explore/logs/#/?_g=(filters:!(),refreshInterval:(pause:!t,value:0),time:(from:now-15m,to:now))&_q=(dataset:(id:d1f424b0-2655-11f1-8baa-d5b726b04d73,timeFieldName:time,title:'logs-otel-v1*',type:INDEX_PATTERN),language:PPL,query:'%7C%20stats%20count()%20as%20log_count%20by%20%60resource.attributes.service.name%60%20%7C%20sort%20-%20log_count')&_a=(legacy:(columns:!(body,severityText,resource.attributes.service.name),interval:auto,isDirty:!f,sort:!()),tab:(logs:(),patterns:(usingRegexPatterns:!f)),ui:(activeTabId:logs,showHistogram:!t))" target="_blank" rel="noopener">Try in playground &rarr;</a>

---

## See also

- [head](/docs/ppl/commands/) -- limit the number of returned results
- [stats](/docs/ppl/commands/stats/) -- aggregate before sorting for better performance
- [eval](/docs/ppl/commands/eval/) -- compute fields to sort by
- [where](/docs/ppl/commands/) -- filter before sorting
