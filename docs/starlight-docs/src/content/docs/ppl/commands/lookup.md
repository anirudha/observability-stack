---
title: "lookup"
description: "Enrich events with data from a lookup index — add context like team names, environment labels, or cost data."
---

import { Tabs, TabItem, Aside } from '@astrojs/starlight/components';

<Aside type="caution">
**Experimental** since OpenSearch 3.0 — syntax may change based on community feedback.
</Aside>

The `lookup` command enriches your search results by matching rows against a reference index (dimension table) and pulling in additional fields. It is the simplest way to add context -- team ownership, environment labels, cost centers, or any static metadata -- to streaming event data.

Compared with `join`, `lookup` is more efficient for one-to-one enrichment against a relatively small, static dataset.

## Syntax

```sql
lookup <lookupIndex> (<lookupMappingField> [AS <sourceMappingField>])...
  [(replace | append | output) (<inputField> [AS <outputField>])...]
```

## Arguments

| Parameter | Required | Description |
|-----------|----------|-------------|
| `<lookupIndex>` | Yes | The name of the lookup index (dimension table) to match against. |
| `<lookupMappingField>` | Yes | A key field in the lookup index used for matching, similar to a join key. Specify multiple fields as a comma-separated list. |
| `<sourceMappingField>` | No | A key field in the source data to match against `lookupMappingField`. Defaults to the same name as `lookupMappingField`. Use `AS` to map differently named fields. |
| `replace \| append \| output` | No | Controls how matched values are applied. Default: `replace`. |
| `<inputField>` | No | A field from the lookup index whose matched value is added to results. If omitted, all non-key fields from the lookup index are applied. |
| `<outputField>` | No | The name of the result field where matched values are placed. Defaults to `inputField`. |

## Output modes

| Mode | Behavior |
|------|----------|
| `replace` | Overwrites existing field values with matched values from the lookup index. If no match is found, the field is set to `null`. This is the default. |
| `append` | Fills only missing (`null`) values in the source data. Existing non-null values are preserved. |
| `output` | Synonym for `replace`. Provided for compatibility. |

## Usage notes

- **Use `lookup` instead of `join`** when enriching events from a small, static reference table. It avoids the overhead of a full join.
- **`replace` overwrites existing values.** If the source data already has a `department` field and the lookup also provides `department`, the lookup value wins. Use `append` if you only want to fill gaps.
- **`append` only fills nulls.** Non-null values in the source data are never overwritten. If the `outputField` does not already exist in the source and you use `append`, the operation fails. Use `replace` to create new fields.
- **Multiple mapping fields** are supported. Separate them with commas to match on a composite key.
- When `<inputField>` is omitted, **all fields** from the lookup index (except the mapping keys) are applied to the output.

## Examples

### Basic lookup — replace values

Enrich worker events with department information from a reference index:

```sql
source = worker
| LOOKUP work_information uid AS id REPLACE department
| fields id, name, occupation, country, salary, department
```

### Append missing values only

Fill in department where it is currently `null`, without overwriting existing values:

```sql
source = worker
| LOOKUP work_information uid AS id APPEND department
| fields id, name, occupation, country, salary, department
```

### Lookup without specifying input fields

When no `inputField` is specified, all non-key fields from the lookup index are applied:

```sql
source = worker
| LOOKUP work_information uid AS id, name
| fields id, name, occupation, country, salary, department
```

### Map to a new output field

Place matched values into a new field using `AS`:

```sql
source = worker
| LOOKUP work_information name REPLACE occupation AS role
| fields id, name, occupation, role
```

### Using the OUTPUT keyword

`OUTPUT` is a synonym for `REPLACE` and produces identical results:

```sql
source = worker
| LOOKUP work_information uid AS id OUTPUT department
| fields id, name, occupation, country, salary, department
```

## Extended examples

### Enrich logs with service ownership

Assume you have a `service_owners` index mapping `service.name` to `team`, `oncall`, and `tier`. Enrich log events with ownership context:

```sql
source = logs-otel-v1*
| eval service = `resource.attributes.service.name`
| LOOKUP service_owners service.name AS service REPLACE team, oncall, tier
| fields time, body, severityText, service, team, oncall, tier
| head 50
```

### Add environment labels to spans

Enrich trace spans with deployment metadata from an `environments` reference index:

```sql
source = otel-v1-apm-span-*
| LOOKUP environments service_name AS serviceName REPLACE env, region, cost_center
| where env = 'production'
| fields serviceName, name, durationInNanos, env, region, cost_center
| sort - durationInNanos
| head 20
```

## See also

- [join](/docs/ppl/commands/join/) — full join for complex multi-field correlation
- [eval](/docs/ppl/commands/) — compute new fields from expressions
- [Command Reference](/docs/ppl/commands/) — all PPL commands
