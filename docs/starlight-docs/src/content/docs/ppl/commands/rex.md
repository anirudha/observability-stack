---
title: "rex"
description: "Extract or substitute fields using regex — with support for sed-mode text replacement and multiple matches."
---

import { Aside } from '@astrojs/starlight/components';

The `rex` command is a more powerful alternative to `parse` for extracting fields from text using Java regular expressions. In addition to standard extraction, `rex` supports **sed mode** for text substitution, **multiple match extraction**, and **offset tracking** to record match positions.

<Aside type="caution">
**Experimental** since OpenSearch 3.3. Parameters may change based on community feedback.
</Aside>

## Syntax

```sql
rex [mode=<mode>] field=<field> <pattern> [max_match=<int>] [offset_field=<string>]
```

## Arguments

| Argument | Required | Default | Description |
|----------|----------|---------|-------------|
| `field` | Yes | -- | The text field to extract data from. Must be a string field. |
| `<pattern>` | Yes | -- | In **extract** mode: a Java regex with named capture groups `(?<name>pattern)`. Group names must start with a letter and contain only letters and digits (no underscores). In **sed** mode: a sed-style pattern (see [Sed mode syntax](#sed-mode-syntax)). |
| `mode` | No | `extract` | `extract` creates new fields from named capture groups. `sed` performs text substitution on the field in place. |
| `max_match` | No | `1` | Maximum number of matches to extract. When greater than 1, extracted fields are returned as arrays. Set to `0` for unlimited matches (capped by the configured system limit, default `10`). |
| `offset_field` | No | -- | Valid in `extract` mode only. Name of an output field that records the character offset positions of each match. |

### Sed mode syntax

In sed mode, the pattern uses one of the following forms:

| Syntax | Description |
|--------|-------------|
| `s/<regex>/<replacement>/` | Substitute the first match of `<regex>` with `<replacement>`. |
| `s/<regex>/<replacement>/g` | Substitute all matches (global flag). |
| `y/<from_chars>/<to_chars>/` | Transliterate characters (like `tr`). |

Backreferences (`\1`, `\2`, etc.) are supported in the replacement string.

### rex vs. parse

| Feature | `rex` | `parse` |
|---------|-------|---------|
| Named capture groups | Yes | Yes |
| Multiple named groups per pattern | Yes | No |
| Multiple matches (`max_match`) | Yes | No |
| Text substitution (sed mode) | Yes | No |
| Offset tracking | Yes | No |

## Usage notes

- In extract mode, each named capture group creates a new string field. When `max_match > 1`, fields become arrays.
- Group names cannot contain underscores or special characters due to Java regex limitations. Use `(?<username>...)` not `(?<user_name>...)`.
- Non-matching patterns return `null` for the extracted fields (unlike `parse` which returns empty strings).
- Multiple `rex` commands can be chained to extract from different fields in the same query.
- The `max_match` system limit defaults to `10` and can be configured via the `plugins.ppl.rex.max_match.limit` cluster setting. Requesting more than the limit results in an error.

## Basic examples

### Extract username and domain from email

Use two named capture groups to split an email address:

```sql
source = accounts
| rex field=email "(?<username>[^@]+)@(?<domain>[^.]+)"
| fields email, username, domain
| head 2
```

| email | username | domain |
|-------|----------|--------|
| amberduke@pyrami.com | amberduke | pyrami |
| hattiebond@netagy.com | hattiebond | netagy |

<a href="https://observability.playground.opensearch.org/w/19jD-R/app/explore/logs/#/?_g=(filters:!(),refreshInterval:(pause:!t,value:0),time:(from:now-15m,to:now))&_q=(dataset:(id:d1f424b0-2655-11f1-8baa-d5b726b04d73,timeFieldName:time,title:'logs-otel-v1*',type:INDEX_PATTERN),language:PPL,query:'source%20%3D%20logs-otel-v1*%20%7C%20rex%20field%3Dbody%20%22(%3F%3Cusername%3E%5B%5E%40%5D%2B)%40(%3F%3Cdomain%3E%5B%5E.%5D%2B)%22%20%7C%20fields%20body%2C%20username%2C%20domain%20%7C%20head%202')&_a=(legacy:(columns:!(body,severityText,resource.attributes.service.name),interval:auto,isDirty:!f,sort:!()),tab:(logs:(),patterns:(usingRegexPatterns:!f)),ui:(activeTabId:logs,showHistogram:!t))" target="_blank" rel="noopener">Try in playground &rarr;</a>

### Replace text using sed mode

Mask email domains by substituting everything after `@`:

```sql
source = accounts
| rex field=email mode=sed "s/@.*/@company.com/"
| fields email
| head 2
```

| email |
|-------|
| amberduke@company.com |
| hattiebond@company.com |

### Extract multiple words with max_match

Pull out multiple words from an address field as an array:

```sql
source = accounts
| rex field=address "(?<words>[A-Za-z]+)" max_match=2
| fields address, words
| head 3
```

| address | words |
|---------|-------|
| 880 Holmes Lane | [Holmes,Lane] |
| 671 Bristol Street | [Bristol,Street] |
| 789 Madison Street | [Madison,Street] |

### Track match positions with offset_field

Record where each capture group matched within the source string:

```sql
source = accounts
| rex field=email "(?<username>[^@]+)@(?<domain>[^.]+)" offset_field=matchpos
| fields email, username, domain, matchpos
| head 2
```

| email | username | domain | matchpos |
|-------|----------|--------|----------|
| amberduke@pyrami.com | amberduke | pyrami | domain=10-15&username=0-8 |
| hattiebond@netagy.com | hattiebond | netagy | domain=11-16&username=0-9 |

### Extract full email components

Capture user, domain, and top-level domain in a single pattern:

```sql
source = accounts
| rex field=email "(?<user>[a-zA-Z0-9._%+-]+)@(?<domain>[a-zA-Z0-9.-]+)\\.(?<tld>[a-zA-Z]{2,})"
| fields email, user, domain, tld
| head 2
```

| email | user | domain | tld |
|-------|------|--------|-----|
| amberduke@pyrami.com | amberduke | pyrami | com |
| hattiebond@netagy.com | hattiebond | netagy | com |

## Extended examples

### Extract structured data from OTel log bodies

Use `rex` to pull out key fields from OpenTelemetry log messages, taking advantage of multiple named groups:

```sql
source = logs-otel-v1*
| rex field=body "(?<method>GET|POST|PUT|DELETE|PATCH)\s+(?<path>/[^\s]*)\s+.*\s(?<status>\d{3})"
| where isnotnull(method)
| stats count() as requests,
        sum(case(cast(status as int) >= 400, 1 else 0)) as errors
  by method, path
| eval error_rate = if(requests > 0, errors * 100.0 / requests, 0)
| sort - requests
| head 20
```

This extracts HTTP method, path, and status code from log bodies, then calculates error rates per endpoint.

<a href="https://observability.playground.opensearch.org/w/19jD-R/app/explore/logs/#/?_g=(filters:!(),refreshInterval:(pause:!t,value:0),time:(from:now-15m,to:now))&_q=(dataset:(id:d1f424b0-2655-11f1-8baa-d5b726b04d73,timeFieldName:time,title:'logs-otel-v1*',type:INDEX_PATTERN),language:PPL,query:'source%20%3D%20logs-otel-v1*%20%7C%20rex%20field%3Dbody%20%22(%3F%3Cmethod%3EGET%7CPOST%7CPUT%7CDELETE%7CPATCH)%5Cs%2B(%3F%3Cpath%3E%2F%5B%5E%5Cs%5D*)%5Cs%2B.*%5Cs(%3F%3Cstatus%3E%5Cd%7B3%7D)%22%20%7C%20where%20isnotnull(method)%20%7C%20stats%20count()%20as%20requests%20by%20method%2C%20path%20%7C%20sort%20-%20requests%20%7C%20head%2020')&_a=(legacy:(columns:!(body,severityText,resource.attributes.service.name),interval:auto,isDirty:!f,sort:!()),tab:(logs:(),patterns:(usingRegexPatterns:!f)),ui:(activeTabId:logs,showHistogram:!t))" target="_blank" rel="noopener">Try in playground &rarr;</a>

### Chain rex commands to extract from multiple fields

Extract initials from separate name fields in a single query:

```sql
source = accounts
| rex field=firstname "(?<firstinitial>^.)"
| rex field=lastname "(?<lastinitial>^.)"
| fields firstname, lastname, firstinitial, lastinitial
| head 3
```

| firstname | lastname | firstinitial | lastinitial |
|-----------|----------|-------------|-------------|
| Amber | Duke | A | D |
| Hattie | Bond | H | B |
| Nanette | Bates | N | B |

<Aside type="caution">
Capture group names **cannot contain underscores**. Use `(?<username>...)` instead of `(?<user_name>...)`. This is a Java regex limitation.
</Aside>

## See also

- [parse](/docs/ppl/commands/parse/) -- simpler regex extraction when you need a single capture group
- [grok](/docs/ppl/commands/grok/) -- extract fields using predefined grok patterns for common formats
- [patterns](/docs/ppl/commands/patterns/) -- automatically discover log patterns without writing regex
- [PPL Functions Reference](/docs/ppl/functions/) -- `regexp_match` and other string functions for regex filtering
