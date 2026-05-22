# 1Password Community Source

Query 1Password vaults, items, and secret details through Coral
SQL using the [1Password Connect Server API](https://developer.1password.com/docs/connect).

## Prerequisites

This source requires a **1Password Connect Server** — a self-hosted
REST API that proxies requests to your 1Password account. It does not
connect directly to the 1Password cloud service.

### 1. Deploy the Connect Server

Follow the official guide to deploy the Connect Server using Docker:

```bash
docker run -d \
  --name op-connect-api \
  -p 8080:8080 \
  -v /path/to/1password-credentials.json:/home/opuser/.op/1password-credentials.json \
  1password/connect-api:latest

docker run -d \
  --name op-connect-sync \
  -v /path/to/1password-credentials.json:/home/opuser/.op/1password-credentials.json \
  1password/connect-sync:latest
```

See [Deploy 1Password Connect](https://developer.1password.com/docs/connect/get-started)
for full instructions including Kubernetes and Docker Compose options.

### 2. Create an access token

In your 1Password account:

1. Go to **Integrations** > **Directory** > **1Password Connect**
2. Select your Connect Server
3. Click **Create Token** and grant it read access to the vaults you need
4. Copy the token

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

-- List all login credentials in a vault
SELECT id, title, category, updated_at
FROM onepassword.items
WHERE vault_id = '<vault-uuid>'
  AND category = 'LOGIN'
ORDER BY updated_at DESC;

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

## Limitations

- **Read-only.** This source does not create, update, or delete any
  1Password resources.
- **Connect Server required.** This source connects to the 1Password
  Connect Server REST API, not the 1Password cloud service directly.
  You must deploy and maintain a Connect Server.
- **No pagination.** The Connect API returns all vaults and items in a
  single response. For accounts with very large numbers of items this
  may result in large payloads.
- **No server-side filtering.** The Connect API supports limited
  SCIM-style filtering by vault name and item title, but does not
  support filtering by category, tags, or dates. All filtering is done
  client-side by Coral after fetching the full response.
- **Sensitive data exposure.** The `item_details` table exposes
  decrypted secret values in the `fields` column. Ensure your Connect
  token has appropriately scoped vault access.

## Out of scope for v1

- Item create/update/delete operations
- File attachment download
- Vault create/update/delete operations
- Events API integration (audit logs, sign-in attempts)
- SCIM user provisioning
