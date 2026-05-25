# Stack Overflow (stackoverflow)

**Version:** 0.1.0
**Backend:** HTTP
**Tables:** 4
**Base URL:** `https://api.stackexchange.com/2.3`

Query Stack Overflow questions, answers, search results, and tags via the
public [Stack Exchange API v2.3](https://api.stackexchange.com/docs). Works
without authentication. An optional API key raises the daily request quota
from 300 to 10 000.

```bash
coral source add --file sources/community/stackoverflow/manifest.yaml
```

## Configuration

| Input                     | Kind       | Required | Default         | Description                                                  |
| ------------------------- | ---------- | -------- | --------------- | ------------------------------------------------------------ |
| `STACKOVERFLOW_API_KEY`   | variable   | no       | *(empty)*       | Stack Exchange API key for higher quota                      |
| `STACKOVERFLOW_SITE`      | variable   | no       | `stackoverflow` | Site to query (`serverfault`, `askubuntu`, `superuser`, etc.) |

To obtain an API key, register an application at
<https://stackapps.com/apps/oauth/register>. Authentication is not required
for read-only access — the key only increases your daily request quota.

## Tables

| Table                       | Description                                    | Key filters               |
| --------------------------- | ---------------------------------------------- | ------------------------- |
| `stackoverflow.questions`   | Browse questions by activity, votes, or tags   | `tagged`, `sort`          |
| `stackoverflow.search`      | Search questions by title keyword              | `intitle` (**required**)  |
| `stackoverflow.answers`     | Recent answers with score and acceptance status | —                         |
| `stackoverflow.tags`        | Tags sorted by popularity with question counts | —                         |

## Example queries

```sql
-- Recent questions sorted by activity
SELECT question_id, title, score, view_count, answer_count
FROM stackoverflow.questions
LIMIT 10;

-- Questions tagged with 'rust'
SELECT question_id, title, score, answer_count, creation_date
FROM stackoverflow.questions
WHERE tagged = 'rust'
LIMIT 10;

-- Questions tagged with both 'python' AND 'django'
SELECT question_id, title, score
FROM stackoverflow.questions
WHERE tagged = 'python;django'
LIMIT 10;

-- Search questions by title keyword
SELECT question_id, title, score, view_count
FROM stackoverflow.search
WHERE intitle = 'async await'
LIMIT 10;

-- Search with tag and sort filters
SELECT question_id, title, score
FROM stackoverflow.search
WHERE intitle = 'dependency injection'
  AND tagged = 'java'
  AND sort = 'votes'
LIMIT 10;

-- Recent answers
SELECT answer_id, question_id, score, is_accepted, creation_date
FROM stackoverflow.answers
LIMIT 10;

-- Most popular tags
SELECT name, count, has_synonyms
FROM stackoverflow.tags
LIMIT 20;
```

## Pagination

All tables use Stack Exchange page-based pagination (default page size 30,
max 100). Coral handles this automatically — just use `LIMIT` to control
how many rows you want.

## Notes

- **No authentication required.** Anonymous access provides 300 API
  requests per day per IP. Adding an API key raises this to 10 000
  per day.
- **Read-only.** This source does not support write operations.
- **HTML-encoded titles.** Question titles may contain HTML entities
  (e.g. `&amp;`, `&#39;`). Use them as-is or decode in your application.
- **Tag AND logic.** The `tagged` filter uses semicolons for AND logic.
  `tagged = 'python;django'` returns questions with **both** tags.
  Passing more than 5 tags always returns zero results.
- **Configurable site.** Set `STACKOVERFLOW_SITE` to query any Stack
  Exchange network site: `serverfault`, `askubuntu`, `superuser`,
  `math`, `unix`, etc.
- **Timestamps.** All date columns are converted from Unix epoch
  seconds to UTC timestamps.
- **Anonymous paging limit.** Without an API key or access token,
  the API limits pagination to page 25 (750 rows at pagesize 30).

## Validation

```bash
coral source lint sources/community/stackoverflow/manifest.yaml
coral source add --file sources/community/stackoverflow/manifest.yaml
coral source test stackoverflow
coral sql "SELECT * FROM coral.tables WHERE schema_name = 'stackoverflow'"

coral sql "SELECT question_id, title, score FROM stackoverflow.questions LIMIT 1"
# +-------------+--------------------------------------------------------------------------+-------+
# | question_id | title                                                                    | score |
# +-------------+--------------------------------------------------------------------------+-------+
# | 79946255    | How to enable &quot;Annotate with Git Blame&quot; using WebStorm 2026.1? | 0     |
# +-------------+--------------------------------------------------------------------------+-------+

coral sql "SELECT question_id, title, score FROM stackoverflow.search WHERE intitle = 'python' LIMIT 1"
# +-------------+-----------------------------------------------+-------+
# | question_id | title                                         | score |
# +-------------+-----------------------------------------------+-------+
# | 72108098    | Sorting words into alphabetic order in Python | -2    |
# +-------------+-----------------------------------------------+-------+

coral sql "SELECT answer_id, score, is_accepted FROM stackoverflow.answers LIMIT 1"
# +-----------+-------+-------------+
# | answer_id | score | is_accepted |
# +-----------+-------+-------------+
# | 53381692  | 76    | false       |
# +-----------+-------+-------------+

coral sql "SELECT name, count FROM stackoverflow.tags LIMIT 1"
# +------------+---------+
# | name       | count   |
# +------------+---------+
# | javascript | 2531304 |
# +------------+---------+
```
