---
title: "grok"
description: "Extract fields using grok patterns — a higher-level alternative to regex with 200+ predefined patterns."
---

import { Aside } from '@astrojs/starlight/components';

The `grok` command parses a text field using grok pattern syntax and appends the extracted fields to the search results. Grok provides over 200 predefined patterns (`%{IP}`, `%{NUMBER}`, `%{HOSTNAME}`, etc.) that wrap common regular expressions, making extraction more readable and less error-prone than writing raw regex.

<Aside type="note">
**Stable** since OpenSearch 2.4.
</Aside>

## Syntax

```sql
grok <field> <grok-pattern>
```

## Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `<field>` | Yes | The text field to parse. |
| `<grok-pattern>` | Yes | A grok pattern using `%{PATTERN:fieldname}` syntax. Each `%{PATTERN:fieldname}` creates a new string field. If a field with the same name already exists, it is overwritten. Raw regex can be mixed with grok patterns. |

## Usage notes

- Grok patterns are built on top of regular expressions but provide a more readable, reusable syntax.
- Use the `%{PATTERN:fieldname}` syntax to extract a named field. If you omit `:fieldname`, the match is consumed but no field is created.
- When parsing a null field, the result is an empty string.
- Grok shares the same [limitations](/docs/ppl/commands/parse/#limitations) as the `parse` command.

### Commonly used grok patterns

| Pattern | Matches | Example |
|---------|---------|---------|
| `%{IP:ip}` | IPv4 or IPv6 address | `192.168.1.1` |
| `%{NUMBER:num}` | Integer or floating-point number | `42`, `3.14` |
| `%{WORD:word}` | Single word (no whitespace) | `ERROR` |
| `%{HOSTNAME:host}` | Hostname or FQDN | `api.example.com` |
| `%{GREEDYDATA:msg}` | Everything (greedy match) | any remaining text |
| `%{HTTPDATE:ts}` | Common log format timestamp | `28/Sep/2022:10:15:57 -0700` |
| `%{IPORHOST:server}` | IP address or hostname | `10.0.0.1` or `web01` |
| `%{COMMONAPACHELOG}` | Full Apache common log line | entire access log entry |
| `%{COMBINEDAPACHELOG}` | Apache combined log line | access log with referrer and user agent |
| `%{SYSLOGLINE}` | Syslog format line | standard syslog entry |
| `%{EMAILADDRESS:email}` | Email address | `user@example.com` |
| `%{URI:url}` | Full URI | `https://example.com/path?q=1` |
| `%{URIPATH:path}` | URI path component | `/api/v1/users` |
| `%{POSINT:code}` | Positive integer | `200`, `404` |
| `%{DATA:val}` | Non-greedy match (minimal) | short text segments |

## Basic examples

### Extract hostname from email addresses

Use the `%{HOSTNAME}` pattern to capture the domain portion of an email:

```sql
source = accounts
| grok email '.+@%{HOSTNAME:host}'
| fields email, host
```

| email | host |
|-------|------|
| amberduke@pyrami.com | pyrami.com |
| hattiebond@netagy.com | netagy.com |
| daleadams@boink.com | boink.com |

<a href="https://observability.playground.opensearch.org/w/19jD-R/app/explore/logs/#/?_g=(filters:!(),refreshInterval:(pause:!t,value:0),time:(from:now-15m,to:now))&_q=(dataset:(id:d1f424b0-2655-11f1-8baa-d5b726b04d73,timeFieldName:time,title:'logs-otel-v1*',type:INDEX_PATTERN),language:PPL,query:'source%20%3D%20logs-otel-v1*%20%7C%20grok%20body%20%27.%2B%40%25%7BHOSTNAME%3Ahost%7D%27%20%7C%20fields%20body%2C%20host')&_a=(legacy:(columns:!(body,severityText,resource.attributes.service.name),interval:auto,isDirty:!f,sort:!()),tab:(logs:(),patterns:(usingRegexPatterns:!f)),ui:(activeTabId:logs,showHistogram:!t))" target="_blank" rel="noopener">Try in playground &rarr;</a>

### Override an existing field

Strip the street number from an address field, keeping only the street name:

```sql
source = accounts
| grok address '%{NUMBER} %{GREEDYDATA:address}'
| fields address
```

| address |
|---------|
| Holmes Lane |
| Bristol Street |
| Madison Street |
| Hutchinson Court |

### Parse Apache-style log lines

Use the built-in `%{COMMONAPACHELOG}` pattern to parse an entire access log entry in one step:

```sql
source = apache
| grok message '%{COMMONAPACHELOG}'
| fields COMMONAPACHELOG, timestamp, response, bytes
```

| timestamp | response | bytes |
|-----------|----------|-------|
| 28/Sep/2022:10:15:57 -0700 | 404 | 19927 |
| 28/Sep/2022:10:15:57 -0700 | 100 | 28722 |
| 28/Sep/2022:10:15:57 -0700 | 401 | 27439 |
| 28/Sep/2022:10:15:57 -0700 | 301 | 9481 |

### Extract IP address and response code from logs

Combine multiple grok patterns to parse custom log formats:

```sql
source = weblogs
| grok message '%{IP:client_ip} .* %{POSINT:status} %{NUMBER:bytes}'
| stats count() as requests, sum(cast(bytes as long)) as total_bytes by status
| sort - requests
```

### Extract numbers and descriptive data

Mix grok patterns with literal text to parse structured output:

```sql
source = accounts
| grok address '%{NUMBER:street_num} %{GREEDYDATA:street_name}'
| where cast(street_num as int) > 500
| fields street_num, street_name
```

| street_num | street_name |
|------------|-------------|
| 671 | Bristol Street |
| 789 | Madison Street |
| 880 | Holmes Lane |

## Extended examples

### Parse OTel log bodies with grok

OpenTelemetry log bodies often contain structured text that grok can parse more readably than raw regex:

```sql
source = logs-otel-v1*
| grok body '%{WORD:level} %{GREEDYDATA:detail}'
| where isnotnull(level)
| stats count() as occurrences by level
| sort - occurrences
```

This extracts the first word from each log body as the log level, then counts occurrences per level.

<a href="https://observability.playground.opensearch.org/w/19jD-R/app/explore/logs/#/?_g=(filters:!(),refreshInterval:(pause:!t,value:0),time:(from:now-15m,to:now))&_q=(dataset:(id:d1f424b0-2655-11f1-8baa-d5b726b04d73,timeFieldName:time,title:'logs-otel-v1*',type:INDEX_PATTERN),language:PPL,query:'source%20%3D%20logs-otel-v1*%20%7C%20grok%20body%20%27%25%7BWORD%3Alevel%7D%20%25%7BGREEDYDATA%3Adetail%7D%27%20%7C%20where%20isnotnull(level)%20%7C%20stats%20count()%20as%20occurrences%20by%20level%20%7C%20sort%20-%20occurrences')&_a=(legacy:(columns:!(body,severityText,resource.attributes.service.name),interval:auto,isDirty:!f,sort:!()),tab:(logs:(),patterns:(usingRegexPatterns:!f)),ui:(activeTabId:logs,showHistogram:!t))" target="_blank" rel="noopener">Try in playground &rarr;</a>

### Extract IP addresses and paths from OTel HTTP logs

Parse HTTP access patterns from log bodies that contain request information:

```sql
source = logs-otel-v1*
| grok body '%{IP:client_ip}.*"%{WORD:method} %{URIPATH:path} HTTP/%{NUMBER:version}" %{POSINT:status}'
| where isnotnull(client_ip)
| stats count() as requests by method, status
| sort - requests
```

<Aside type="tip">
When grok patterns become very long, consider whether `parse` with a targeted regex might be simpler. Grok excels at parsing well-known formats (Apache logs, syslog, etc.); for ad-hoc extraction of one or two fields, `parse` or `rex` may be more concise.
</Aside>

## See also

- [parse](/docs/ppl/commands/parse/) -- extract fields using raw Java regex (more control, less readability)
- [rex](/docs/ppl/commands/rex/) -- regex extraction with sed-mode text replacement and multiple matches
- [patterns](/docs/ppl/commands/patterns/) -- automatically discover log patterns without writing any patterns
