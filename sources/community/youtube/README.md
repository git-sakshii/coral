# YouTube community source

The `youtube` community source exposes read-only YouTube video, channel, playlist, comment, and search data through Coral SQL using the YouTube Data API v3.

## Setup

YouTube Data API v3 requires an API key for public data access. No OAuth is needed.

### 🔑 Get an API Key

1. Open the [Google Cloud Console](https://console.cloud.google.com/).
2. Create a project (or select an existing one).
3. Go to **APIs & Services** → **Library**.
4. Search for **YouTube Data API v3** and click **Enable**.
5. Go to **Credentials** → **Create Credentials** → **API Key**.
6. Copy the generated API key.

> [!NOTE]
> The free tier provides **10,000 quota units per day**. Most requests cost 1 unit, but `search` costs 100 units per request. See [YouTube API quota](https://developers.google.com/youtube/v3/determine_quota_cost) for details.

### 🚀 Connect

```sh
export YOUTUBE_API_KEY="<your-api-key>"
coral source add --file sources/community/youtube/manifest.yaml
```

## Tables

| Table | Purpose | Required Filters |
| --- | --- | --- |
| `youtube.videos` | Video metadata, statistics, and content details | `id` or `chart` |
| `youtube.channels` | Channel metadata and statistics | `id` or `for_handle` |
| `youtube.playlists` | Playlist metadata for a channel | `id` or `channel_id` |
| `youtube.playlist_items` | Videos inside a specific playlist | `playlist_id` |
| `youtube.search` | Search YouTube for videos, channels, or playlists | `q` |
| `youtube.comment_threads` | Top-level comments on a video | `video_id` |
| `youtube.video_categories` | Video category definitions | None (defaults to US region) |

All tables are read-only. This source does not upload, modify, or delete any YouTube content.

### Important Design Quirks

* **Quota Costs**: The `search` table costs 100 quota units per request (vs 1 unit for `videos`, `channels`, etc.). Use it sparingly to stay within the 10,000 daily quota.
* **Statistics as Strings**: The YouTube API returns view/like/comment counts as strings, not integers. Use `CAST(view_count AS BIGINT)` in SQL if you need numeric operations.
* **Duration Format**: Video duration is returned as an ISO 8601 string (e.g. `PT4M13S` = 4 minutes 13 seconds), not as a number of seconds.
* **Search Result IDs**: The `id` column in the search table contains the video/channel/playlist ID depending on the result `kind`. Use the `kind` column to distinguish between result types.
* **Pagination Limits**: All tables support cursor-based pagination via `nextPageToken`. Maximum page size is 50 items for most endpoints and 100 for `comment_threads`.

## Example queries

Look up metadata for a specific video:

```sql
SELECT id, title, channel_title, view_count, like_count, duration
FROM youtube.videos
WHERE id = 'dQw4w9WgXcQ';
```

Get channel statistics by handle:

```sql
SELECT id, title, subscriber_count, video_count, view_count
FROM youtube.channels
WHERE for_handle = '@Google';
```

Search for videos about a topic:

```sql
SELECT id, title, channel_title, published_at
FROM youtube.search
WHERE q = 'rust programming'
  AND type = 'video'
  AND order = 'viewCount'
LIMIT 10;
```

List playlists for a channel:

```sql
SELECT id, title, item_count, published_at
FROM youtube.playlists
WHERE channel_id = 'UC_x5XG1OV2P6uZZ5FSM9Ttw'
LIMIT 20;
```

List videos in a playlist:

```sql
SELECT position, title, video_id, video_published_at
FROM youtube.playlist_items
WHERE playlist_id = 'PLIivdWyY5sqJxnwJhe3ETaK46_a2PARsN'
ORDER BY position
LIMIT 50;
```

Read comments on a video:

```sql
SELECT author_display_name, text_original, like_count, published_at
FROM youtube.comment_threads
WHERE video_id = 'dQw4w9WgXcQ'
  AND order = 'relevance'
LIMIT 20;
```

Browse video categories for a region:

```sql
SELECT id, title, assignable
FROM youtube.video_categories
WHERE region_code = 'US';
```

## Validation

Lint the manifest:

```sh
coral source lint sources/community/youtube/manifest.yaml
```

Install and test with your API key:

```sh
export YOUTUBE_API_KEY="<your-api-key>"
coral source add --file sources/community/youtube/manifest.yaml
coral source test youtube
```

Inspect the registered source metadata:

```sh
coral sql "SELECT table_name, description FROM coral.tables WHERE schema_name = 'youtube'"
coral sql "SELECT table_name, column_name, data_type FROM coral.columns WHERE schema_name = 'youtube' ORDER BY table_name, ordinal_position"
```
