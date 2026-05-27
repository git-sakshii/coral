# rubygems

Query the [RubyGems.org](https://rubygems.org) package registry — the standard
hosting service for Ruby gems — using SQL. Search gems by keyword, inspect version
history, and monitor newly published packages. No API key or account required.

---

## Authentication

None. All tables use fully public RubyGems.org API endpoints.

---

## Installation

Clone the Coral repository if you have not already:

```bash
git clone https://github.com/withcoral/coral.git
cd coral
```

Add the source from the manifest file:

```bash
coral source add --file sources/community/rubygems/manifest.yaml
```

---

## Tables

| Table          | Description                                      | Required Filter              |
|----------------|--------------------------------------------------|------------------------------|
| `search`       | Search gems by keyword, one row per gem          | `WHERE query = '...'`        |
| `gem`          | Full details for a single gem by exact name      | `WHERE gem_name = '...'`     |
| `versions`     | All published versions of a gem                  | `WHERE gem_name = '...'`     |
| `latest`       | 50 gems most recently added to RubyGems.org      | None                         |
| `just_updated` | 50 gems most recently updated on RubyGems.org    | None                         |

---

## Example Queries

```sql
-- Search for testing gems
SELECT name, version, downloads, authors, licenses
FROM rubygems.search
WHERE query = 'testing'
ORDER BY downloads DESC
LIMIT 10;

-- Full details for a known gem
SELECT name, version, downloads, info, source_code_uri, bug_tracker_uri
FROM rubygems.gem
WHERE gem_name = 'rails';

-- Version history for a gem, newest first
SELECT number, platform, prerelease, downloads_count, created_at
FROM rubygems.versions
WHERE gem_name = 'rails'
ORDER BY created_at DESC
LIMIT 20;

-- Find prerelease versions only
SELECT number, platform, downloads_count, created_at
FROM rubygems.versions
WHERE gem_name = 'rails' AND prerelease = true
ORDER BY created_at DESC;

-- What gems were just published?
SELECT name, version, authors, info
FROM rubygems.latest;

-- What was updated recently?
SELECT name, version, downloads, info
FROM rubygems.just_updated;

-- Find gems with MIT license
SELECT name, version, downloads, licenses
FROM rubygems.search
WHERE query = 'authentication'
ORDER BY downloads DESC
LIMIT 10;

-- Join with GitHub to see issues for a gem's repo
-- (requires github source also added)
SELECT g.name, g.version, i.title, i.state
FROM rubygems.gem g
JOIN github.issues i
  ON i.owner = 'rails' AND i.repo = 'rails' AND i.state = 'open'
WHERE g.gem_name = 'rails'
LIMIT 10;
```

---

## Notes

- The `search` table returns 30 gems per page. Use `LIMIT` and `OFFSET` for
  pagination or add more specific keywords to narrow results.
- The `versions` table returns all versions at once with no pagination.
  Well-established gems like `rails` may have 100+ versions.
- The `gem` and `versions` tables require an exact gem name — use `search` first
  to discover the right name if unsure.
- The `latest` and `just_updated` tables return at most 50 rows each.
- RubyGems.org enforces rate limits. Avoid tight query loops across many gems.
