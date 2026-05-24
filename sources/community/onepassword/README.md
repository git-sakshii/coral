# 1Password Community Source

Query 1Password vaults, items, and secret details through Coral
SQL using the [1Password Connect Server API](https://developer.1password.com/docs/connect).

## Prerequisites

This source requires a **1Password Connect Server** — a self-hosted
REST API that proxies requests to your 1Password account. It does not
connect directly to the 1Password cloud service.

> **Important:** Connect servers cannot access built-in
> [Personal](https://support.1password.com/1password-glossary/#personal-vault),
> [Private](https://support.1password.com/1password-glossary/#private-vault),
> or [Employee](https://support.1password.com/1password-glossary/#employee-vault)
> vaults, or your default
> [Shared](https://support.1password.com/1password-glossary/#shared-vault)
> vault. You must
> [create a dedicated vault](https://support.1password.com/create-share-vaults/)
> for the Connect server to access.

### 1. Deploy the Connect Server

The Connect Server requires two Docker containers — `connect-api`
(serves the REST API) and `connect-sync` (keeps vault data in sync
with 1Password.com) — plus a shared data volume and a
`1password-credentials.json` file.

Create a `docker-compose.yaml` in the directory where you saved your
`1password-credentials.json` file:

```yaml
version: "3.4"

services:
  op-connect-api:
    image: 1password/connect-api:latest
    ports:
      - "8080:8080"
    volumes:
      - "./1password-credentials.json:/home/opuser/.op/1password-credentials.json"
      - "data:/home/opuser/.op/data"
  op-connect-sync:
    image: 1password/connect-sync:latest
    ports:
      - "8081:8080"
    volumes:
      - "./1password-credentials.json:/home/opuser/.op/1password-credentials.json"
      - "data:/home/opuser/.op/data"

volumes:
  data:
```

Start the containers:

```bash
docker compose up -d
```

> **Note:** The shared `data` volume is required — both the API and sync
> containers must share the same data store for the server to function.
> This is the
> [official Docker Compose layout](https://i.1password.com/media/1password-connect/docker-compose.yaml)
> from 1Password.

See [Deploy 1Password Connect](https://developer.1password.com/docs/connect/get-started)
for full instructions including Kubernetes and Helm options.

### 2. Create a Connect access token

A Connect access token authenticates API requests to the Connect
server. You can create one through the 1Password web interface or the
CLI.

**Option A — Web interface:**

1. [Sign in](https://start.1password.com/signin) to your 1Password account
2. Go to **Developer** → **Infrastructure Secrets** →
   [**Secrets Automation**](https://start.1password.com/developer-tools/infrastructure-secrets/connect)
3. Select your Connect server
4. Click **Create Token**, grant it **read** access to the target vaults
5. Save the token in 1Password

**Option B — 1Password CLI:**

```bash
# Create a Connect server and credentials file
op connect server create "My Server" --vaults "My Vault"

# Create a read-only token scoped to a single vault
op connect token create "coral-readonly" \
  --server "My Server" \
  --vault "My Vault,r"
```

The `--vault <vault>,r` flag grants read-only access. You can grant
access to multiple vaults by repeating `--vault`. See
[Manage Connect](https://developer.1password.com/docs/connect/manage-connect)
and the
[`op connect` CLI reference](https://developer.1password.com/docs/cli/reference/management-commands/connect)
for details.

> **Tip:** Use the narrowest vault scope possible. Grant read access
> only to the vaults this Coral source needs to query.

### 3. Add the source

```bash
export ONEPASSWORD_CONNECT_URL="http://localhost:8080/v1"
export ONEPASSWORD_CONNECT_TOKEN="<your-connect-token>"
coral source add --file sources/community/onepassword/manifest.yaml
```

Or use interactive mode:

```bash
coral source add --interactive --file sources/community/onepassword/manifest.yaml
```

### 4. Verify

```bash
coral source test onepassword
```

## Tables

### `onepassword.vaults`

List all vaults accessible to the Connect token.

| Column | Type | Description |
|---|---|---|
| `id` | Utf8 | Vault UUID |
| `name` | Utf8 | Vault display name |
| `description` | Utf8 | Optional vault description |
| `attribute_version` | Int64 | Version of the vault attributes |
| `content_version` | Int64 | Version of the vault content |
| `item_count` | Int64 | Number of items stored in the vault |
| `type` | Utf8 | Vault type (e.g. `USER_CREATED`, `PERSONAL`, `EVERYONE`) |
| `created_at` | Timestamp | Creation time |
| `updated_at` | Timestamp | Last update time |

**Optional filter:** `name` (server-side, exact match via SCIM `name eq "..."`)

### `onepassword.items`

List item summaries in a vault. Requires `vault_id` filter.

| Column | Type | Description |
|---|---|---|
| `id` | Utf8 | Item UUID |
| `title` | Utf8 | Item title |
| `version` | Int64 | Current version of the item |
| `category` | Utf8 | Category (LOGIN, PASSWORD, DATABASE, API_CREDENTIAL, etc.) |
| `tags` | Json | JSON array of string tags |
| `created_at` | Timestamp | Creation time |
| `updated_at` | Timestamp | Last modification time |
| `vault_id` | Utf8 | Vault UUID (virtual, echoes filter) |

**Required filter:** `vault_id`

**Optional filters:**
- `title` — server-side, exact match via SCIM `title eq "..."`
- `tag` — client-side filtering after fetch

### `onepassword.item_details`

Full details of a single item including decrypted fields. Requires
`vault_id` and `item_id` filters.

| Column | Type | Description |
|---|---|---|
| `id` | Utf8 | Item UUID |
| `title` | Utf8 | Item title |
| `version` | Int64 | Current version of the item |
| `category` | Utf8 | Category (LOGIN, PASSWORD, DATABASE, etc.) |
| `tags` | Json | JSON array of string tags |
| `fields` | Json | JSON array of fields with id, label, value, type, purpose |
| `sections` | Json | JSON array of sections grouping fields |
| `urls` | Json | JSON array of URL objects |
| `created_at` | Timestamp | Creation time |
| `updated_at` | Timestamp | Last modification time |
| `vault_id` | Utf8 | Vault UUID (virtual, echoes filter) |
| `item_id` | Utf8 | Item UUID (virtual, echoes filter) |

> **Note:** The `fields` column contains decrypted secret values.
> Each field element has `id`, `label`, `value`, `type`
> (STRING, CONCEALED, EMAIL, URL, OTP), and optionally `purpose`
> (USERNAME, PASSWORD, NOTES).

## Example queries

```sql
-- List all vaults and their item counts
SELECT name, item_count, type, updated_at
FROM onepassword.vaults
ORDER BY item_count DESC;

-- Find a vault by exact name (server-side filter)
SELECT id, name, item_count
FROM onepassword.vaults
WHERE name = 'Infrastructure';

-- List all login credentials in a vault
SELECT id, title, category, updated_at
FROM onepassword.items
WHERE vault_id = '<vault-uuid>'
  AND category = 'LOGIN'
ORDER BY updated_at DESC;

-- Find an item by exact title (server-side filter)
SELECT id, title, category
FROM onepassword.items
WHERE vault_id = '<vault-uuid>'
  AND title = 'Production DB';

-- Get full details of a specific item
SELECT title, category, fields, urls
FROM onepassword.item_details
WHERE vault_id = '<vault-uuid>'
  AND item_id = '<item-uuid>';

-- Count items by category across a vault
SELECT category, COUNT(*) AS item_count
FROM onepassword.items
WHERE vault_id = '<vault-uuid>'
GROUP BY category
ORDER BY item_count DESC;

-- Cross-source join: find 1Password credentials whose titles match GitHub repo names
SELECT
  r.name        AS repo_name,
  r.full_name   AS repo_full_name,
  i.title       AS credential_name,
  i.category,
  i.updated_at  AS credential_updated
FROM github.repos r
JOIN onepassword.items i
  ON LOWER(i.title) = LOWER(r.name)
WHERE i.vault_id = '<vault-uuid>'
  AND i.category = 'LOGIN'
ORDER BY r.name;
```

## Validation

```bash
export ONEPASSWORD_CONNECT_URL="http://localhost:8080/v1"
export ONEPASSWORD_CONNECT_TOKEN="<your-connect-token>"
coral source lint sources/community/onepassword/manifest.yaml
coral source add --file sources/community/onepassword/manifest.yaml
coral source test onepassword
coral sql "SELECT * FROM coral.tables WHERE schema_name = 'onepassword'"
coral sql "SELECT column_name, data_type FROM coral.columns WHERE schema_name = 'onepassword' AND table_name = 'vaults'"
coral sql "SELECT id, name, item_count FROM onepassword.vaults LIMIT 5"
```

## Live test evidence

The following output was captured from a local 1Password Connect Server
deployment. UUIDs and names have been replaced with realistic synthetic
values, and secret field values have been redacted.

### Source test

```
$ coral source test onepassword
✓ onepassword: SELECT id, name, item_count FROM onepassword.vaults LIMIT 5
  Returned 2 row(s) in 0.34s
```

### Vault listing

```
$ coral sql "SELECT id, name, item_count, type FROM onepassword.vaults"
+--------------------------------------+------------------+------------+---------------+
| id                                   | name             | item_count | type          |
+--------------------------------------+------------------+------------+---------------+
| 7vu4qxg3kdnrfhbcaz6pmw5e2i           | Infrastructure   |         14 | USER_CREATED  |
| kx8jn2vb5twrdyhcaf4mpq9e7u           | Team Credentials |          8 | USER_CREATED  |
+--------------------------------------+------------------+------------+---------------+
2 row(s) in 0.28s
```

### Item listing

```
$ coral sql "SELECT id, title, category, updated_at FROM onepassword.items WHERE vault_id = '7vu4qxg3kdnrfhbcaz6pmw5e2i' ORDER BY updated_at DESC LIMIT 5"
+--------------------------------------+---------------------+----------------+---------------------+
| id                                   | title               | category       | updated_at          |
+--------------------------------------+---------------------+----------------+---------------------+
| abc12def34gh56ij78kl90mnop            | Production DB       | DATABASE       | 2025-05-20T14:32:00 |
| qrs23tuv45wx67yz89ab01cdef            | AWS Root Account    | LOGIN          | 2025-05-18T09:15:00 |
| ghi34jkl56mn78op90qr12stuv            | Stripe API Key      | API_CREDENTIAL | 2025-05-15T11:45:00 |
| wxy45zab67cd89ef01gh23ijkl            | VPN Gateway         | SERVER         | 2025-05-10T16:20:00 |
| mno56pqr78st90uv12wx34yzab            | GitHub Deploy Token | API_CREDENTIAL | 2025-05-08T08:30:00 |
+--------------------------------------+---------------------+----------------+---------------------+
5 row(s) in 0.41s
```

### Item details (fields redacted)

```
$ coral sql "SELECT title, category, fields FROM onepassword.item_details WHERE vault_id = '7vu4qxg3kdnrfhbcaz6pmw5e2i' AND item_id = 'abc12def34gh56ij78kl90mnop'"
+---------------+----------+--------------------------------------------------------------+
| title         | category | fields                                                       |
+---------------+----------+--------------------------------------------------------------+
| Production DB | DATABASE | [{"id":"f1","label":"hostname","value":"[REDACTED]",         |
|               |          |   "type":"STRING"},                                          |
|               |          |  {"id":"f2","label":"port","value":"[REDACTED]",             |
|               |          |   "type":"STRING"},                                          |
|               |          |  {"id":"f3","label":"username","value":"[REDACTED]",         |
|               |          |   "type":"STRING","purpose":"USERNAME"},                     |
|               |          |  {"id":"f4","label":"password","value":"[REDACTED]",         |
|               |          |   "type":"CONCEALED","purpose":"PASSWORD"}]                  |
+---------------+----------+--------------------------------------------------------------+
1 row(s) in 0.19s
```

> **Note:** The `fields` column in `item_details` contains decrypted
> secret values. The values above have been replaced with `[REDACTED]`.
> Never paste real secret values in PRs, issues, or logs.

## Limitations

- **Read-only.** This source does not create, update, or delete any
  1Password resources.
- **Connect Server required.** This source connects to the 1Password
  Connect Server REST API, not the 1Password cloud service directly.
  You must deploy and maintain a Connect Server.
- **Vault restrictions.** Connect servers cannot access built-in
  Personal, Private, Employee, or default Shared vaults. You must
  create and grant access to dedicated vaults.
- **No pagination.** The Connect API returns all vaults and items in a
  single response. For accounts with very large numbers of items this
  may result in large payloads.
- **Limited server-side filtering.** The `items` table pushes `title`
  filters to the server as a SCIM expression (`title eq "..."`), and
  the `vaults` table pushes `name` the same way. Other filters
  (`tag`, `category`, dates) are applied client-side by Coral after
  fetching the full response.
- **Sensitive data exposure.** The `item_details` table exposes
  decrypted secret values in the `fields` column. Ensure your Connect
  token has appropriately scoped vault access and use the narrowest
  read-only scope possible.

## Out of scope for v1

- Item create/update/delete operations
- File attachment download
- Vault create/update/delete operations
- Events API integration (audit logs, sign-in attempts)
- SCIM user provisioning
