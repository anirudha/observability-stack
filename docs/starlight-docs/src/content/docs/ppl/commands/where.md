---
title: "where"
description: "Filter search results using boolean expressions — the primary filtering command in PPL."
---

## Description

The `where` command filters search results to only those rows where the specified boolean expression evaluates to `true`. It is the **primary filtering command** in PPL and can appear anywhere in the pipeline after the `search` (or `source=`) command.

`where` supports all comparison operators, logical operators, pattern matching with `LIKE`, set membership with `IN`, range checks with `BETWEEN`, null testing with `IS NULL` / `IS NOT NULL`, and nested conditions with parentheses. You can also use built-in functions and `eval` expressions inline within the boolean expression.

## Syntax

```sql
where <boolean-expression>
```

## Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `<boolean-expression>` | Yes | The condition used to filter results. Only rows where this evaluates to `true` are returned. |

### Supported operators

| Category | Operators |
|----------|-----------|
| **Comparison** | `=`, `!=`, `<>`, `>`, `<`, `>=`, `<=` |
| **Logical** | `AND`, `OR`, `NOT` |
| **Pattern matching** | `LIKE(field, pattern)` — `%` matches zero or more characters, `_` matches exactly one character |
| **Set membership** | `IN (value1, value2, ...)` |
| **Range** | `BETWEEN value1 AND value2` |
| **Null testing** | `IS NULL`, `IS NOT NULL`, `ISNULL(field)`, `ISNOTNULL(field)` |
| **Grouping** | Parentheses `( )` for controlling evaluation order |

## Usage notes

- **Multiple where commands**: You can chain multiple `where` commands in a single pipeline. Each successive `where` further narrows the result set, equivalent to combining them with `AND`.
- **Eval expressions inline**: You can use functions and expressions directly in the boolean condition (e.g., `where length(body) > 100` or `where LIKE(body, '%timeout%')`).
- **Null handling**: Comparisons with `null` values follow SQL semantics — a comparison involving `null` evaluates to `null` (not `true` or `false`), so the row is excluded. Use `IS NULL` or `ISNULL()` to explicitly test for null values.
- **String values**: Enclose string literals in single quotes (`'value'`). Double quotes are used for field names that contain special characters.
- **Backtick field names**: OTel fields with dots in their names (e.g., `resource.attributes.service.name`) must be enclosed in backticks to prevent them from being interpreted as nested field access.
- **Performance**: Filters applied earlier in the pipeline reduce the amount of data processed by subsequent commands. Place your most selective `where` conditions as early as possible.
- **vs. search expression**: The `search` command also supports inline boolean expressions, but `where` is more flexible — it supports functions, `LIKE`, `BETWEEN`, and computed expressions that `search` does not.

## Basic examples

### Simple comparison

Return accounts with a balance greater than 30,000:

```sql
source=accounts
| where balance > 30000
| fields account_number, balance
```

### Combine conditions with AND / OR

Return male accounts older than 30:

```sql
source=accounts
| where age > 30 AND gender = 'M'
| fields account_number, age, gender
```

Return accounts where the account number is 1 or gender is female:

```sql
source=accounts
| where account_number = 1 OR gender = 'F'
| fields account_number, gender
```

### Pattern matching with LIKE

Find states starting with `V`:

```sql
source=accounts
| where LIKE(state, 'V%')
| fields account_number, state
```

Find two-letter states starting with `M`:

```sql
source=accounts
| where LIKE(state, 'M_')
| fields account_number, state
```

### Set membership with IN

Return accounts in specific states:

```sql
source=accounts
| where state IN ('IL', 'VA')
| fields account_number, state
```

### Null testing

Find records where a field is null:

```sql
source=accounts
| where ISNULL(employer)
| fields account_number, employer
```

Find records where a field has a value:

```sql
source=accounts
| where ISNOTNULL(email)
| fields account_number, email
```

### Grouped conditions

Combine multiple conditions with parentheses to control evaluation order:

```sql
source=accounts
| where (balance > 40000 OR age > 35) AND gender = 'M'
| fields account_number, balance, age, gender
```

## Extended examples

### Filter error logs by service

Find ERROR and FATAL logs from a specific service using OTel semantic convention fields. This is a common starting point for incident triage.

```sql
source=logs-otel-v1*
| where severityText = 'ERROR' OR severityText = 'FATAL'
| where `resource.attributes.service.name` = 'cart-service'
| fields time, body, severityText, `resource.attributes.service.name`
| head 50
```

<a href="https://observability.playground.opensearch.org/w/19jD-R/app/explore/logs/#/?_g=(filters:!(),refreshInterval:(pause:!t,value:0),time:(from:now-15m,to:now))&_q=(dataset:(id:d1f424b0-2655-11f1-8baa-d5b726b04d73,timeFieldName:time,title:'logs-otel-v1*',type:INDEX_PATTERN),language:PPL,query:'%7C%20where%20severityText%20%3D%20%27ERROR%27%20OR%20severityText%20%3D%20%27FATAL%27%20%7C%20where%20%60resource.attributes.service.name%60%20%3D%20%27cart-service%27%20%7C%20fields%20time%2C%20body%2C%20severityText%2C%20%60resource.attributes.service.name%60%20%7C%20head%2050')&_a=(legacy:(columns:!(body,severityText,resource.attributes.service.name),interval:auto,isDirty:!f,sort:!()),tab:(logs:(),patterns:(usingRegexPatterns:!f)),ui:(activeTabId:logs,showHistogram:!t))" target="_blank" rel="noopener">Try in playground &rarr;</a>

### Compound GenAI attribute filter

Filter logs for a specific GenAI model and operation, useful for investigating LLM call failures or high-latency completions.

```sql
source=logs-otel-v1*
| where `attributes.gen_ai.request.model` = 'anthropic.claude-sonnet-4-20250514'
| where `attributes.gen_ai.operation.name` = 'chat'
| where severityNumber >= 17
| fields time, body, `resource.attributes.service.name`, `attributes.gen_ai.request.model`
| head 20
```

<a href="https://observability.playground.opensearch.org/w/19jD-R/app/explore/logs/#/?_g=(filters:!(),refreshInterval:(pause:!t,value:0),time:(from:now-15m,to:now))&_q=(dataset:(id:d1f424b0-2655-11f1-8baa-d5b726b04d73,timeFieldName:time,title:'logs-otel-v1*',type:INDEX_PATTERN),language:PPL,query:'%7C%20where%20%60attributes.gen_ai.request.model%60%20%3D%20%27anthropic.claude-sonnet-4-20250514%27%20%7C%20where%20%60attributes.gen_ai.operation.name%60%20%3D%20%27chat%27%20%7C%20where%20severityNumber%20%3E%3D%2017%20%7C%20fields%20time%2C%20body%2C%20%60resource.attributes.service.name%60%2C%20%60attributes.gen_ai.request.model%60%20%7C%20head%2020')&_a=(legacy:(columns:!(body,severityText,resource.attributes.service.name),interval:auto,isDirty:!f,sort:!()),tab:(logs:(),patterns:(usingRegexPatterns:!f)),ui:(activeTabId:logs,showHistogram:!t))" target="_blank" rel="noopener">Try in playground &rarr;</a>

### Filter logs containing a keyword pattern

Find logs whose body contains the word "timeout" using `LIKE` with wildcard characters:

```sql
source=logs-otel-v1*
| where LIKE(body, '%timeout%')
| fields time, body, severityText, `resource.attributes.service.name`
| head 20
```

<a href="https://observability.playground.opensearch.org/w/19jD-R/app/explore/logs/#/?_g=(filters:!(),refreshInterval:(pause:!t,value:0),time:(from:now-15m,to:now))&_q=(dataset:(id:d1f424b0-2655-11f1-8baa-d5b726b04d73,timeFieldName:time,title:'logs-otel-v1*',type:INDEX_PATTERN),language:PPL,query:'%7C%20where%20LIKE(body%2C%20%27%25timeout%25%27)%20%7C%20fields%20time%2C%20body%2C%20severityText%2C%20%60resource.attributes.service.name%60%20%7C%20head%2020')&_a=(legacy:(columns:!(body,severityText,resource.attributes.service.name),interval:auto,isDirty:!f,sort:!()),tab:(logs:(),patterns:(usingRegexPatterns:!f)),ui:(activeTabId:logs,showHistogram:!t))" target="_blank" rel="noopener">Try in playground &rarr;</a>

## See also

- [`search`](/docs/ppl/commands/search/) — The starting point of every PPL query, also supports inline boolean expressions
- [`fields`](/docs/ppl/commands/fields/) — Select or exclude specific fields from the output
- [PPL Commands](/docs/ppl/commands/) — Full command reference
