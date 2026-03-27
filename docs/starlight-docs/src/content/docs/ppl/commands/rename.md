---
title: "rename"
description: "Rename fields in search results — simplify long OTel attribute names for readability."
---

import { Tabs, TabItem, Aside } from '@astrojs/starlight/components';

<Aside type="note">
**Stable** since OpenSearch 1.0
</Aside>

The `rename` command renames one or more fields in your search results. It is especially useful for simplifying the long, dot-delimited attribute names common in OpenTelemetry data (e.g., `resource.attributes.service.name`) into shorter, readable aliases.

## Syntax

```sql
rename <source-field> AS <target-field> [, <source-field> AS <target-field>]...
```

## Arguments

| Parameter | Required | Description |
|-----------|----------|-------------|
| `<source-field>` | Yes | The current field name. Supports wildcard patterns using `*`. |
| `<target-field>` | Yes | The new name for the field. Must contain the same number of wildcards as the source. |

## Usage notes

- **Multiple renames** can be specified in a single command, separated by commas.
- **Wildcard patterns** (`*`) match any sequence of characters. Both the source and target must have the same number of wildcards. For example, `*name` matches `firstname` and `lastname`, and renaming to `*_name` produces `first_name` and `last_name`.
- **Renaming to an existing field** removes the original target field and replaces it with the source field's values.
- **Renaming a non-existent field to an existing field** removes the target field from results.
- **Renaming a non-existent field to a non-existent field** has no effect.
- The `rename` command executes on the coordinating node and is **not pushed down** to the query DSL.
- Literal `*` characters in field names cannot be escaped -- the asterisk is always treated as a wildcard.

## Examples

### Rename a single field

```sql
source = accounts
| rename account_number as an
| fields an
```

### Rename multiple fields

```sql
source = accounts
| rename account_number as an, employer as emp
| fields an, emp
```

### Rename with wildcards

Match all fields ending in `name` and add an underscore:

```sql
source = accounts
| rename *name as *_name
| fields first_name, last_name
```

### Multiple wildcard patterns

Combine several wildcard renames in one command:

```sql
source = accounts
| rename *name as *_name, *_number as *number
| fields first_name, last_name, accountnumber
```

### Rename an existing field to another existing field

The target field is replaced by the source field's values:

```sql
source = accounts
| rename firstname as age
| fields age
```

The `age` column now contains the original `firstname` values.

## Extended examples

### Simplify OTel attribute names for log analysis

OpenTelemetry log fields have long, dot-delimited names. Rename them for readability before analysis:

```sql
source = logs-otel-v1*
| rename `resource.attributes.service.name` as service,
         `resource.attributes.service.version` as version,
         `resource.attributes.deployment.environment` as env
| where severityText = 'ERROR'
| stats count() as errors by service, version, env
| sort - errors
```

<a href="https://observability.playground.opensearch.org/w/19jD-R/app/explore/logs/#/?_g=(filters:!(),refreshInterval:(pause:!t,value:0),time:(from:now-15m,to:now))&_q=(dataset:(id:d1f424b0-2655-11f1-8baa-d5b726b04d73,timeFieldName:time,title:'logs-otel-v1*',type:INDEX_PATTERN),language:PPL,query:'%7C%20rename%20%60resource.attributes.service.name%60%20as%20service%20%7C%20fields%20time%2C%20body%2C%20severityText%2C%20service%20%7C%20head%2020')&_a=(legacy:(columns:!(body,severityText,resource.attributes.service.name),interval:auto,isDirty:!f,sort:!()),tab:(logs:(),patterns:(usingRegexPatterns:!f)),ui:(activeTabId:logs,showHistogram:!t))" target="_blank" rel="noopener">Try in playground &rarr;</a>

### Rename span fields for dashboard readability

Shorten trace span attribute names for cleaner output in dashboards:

```sql
source = otel-v1-apm-span-*
| rename serviceName as service, durationInNanos as duration_ns
| eval duration_ms = duration_ns / 1000000
| fields service, name, duration_ms, traceId
| sort - duration_ms
| head 20
```

## See also

- [fields](/docs/ppl/commands/) — select or exclude fields
- [eval](/docs/ppl/commands/) — create computed fields
- [Command Reference](/docs/ppl/commands/) — all PPL commands
