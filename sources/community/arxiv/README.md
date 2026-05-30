# arXiv Source

Search arXiv papers, preprints, and research metadata.

Under the hood, this source queries the open-access [OpenAlex API](https://openalex.org/), which indexes the entire arXiv preprint repository. This approach allows Coral to fetch and parse paper metadata in native JSON format, bypassing the XML-only limitation of the official arXiv API.

## Rate Limits & Usage

OpenAlex is free and open to the public without requiring an API key.
- **Default limit:** 10 requests per second.
- **Polite pool:** If you wish to identify your application and receive faster, prioritized rate limits (up to 100 requests per second), OpenAlex recommends adding a `mailto` parameter to requests. Since this community source uses a fully anonymous setup, you can optionally modify the manifest's `base_url` to append your email, e.g. `https://api.openalex.org?mailto=your-email@example.com`.

## Setup

Add the source directly using the CLI:

```bash
coral source add --file sources/community/arxiv/manifest.yaml
```

## Functions

### `search(q)`
Perform a full-text search across arXiv preprint titles, abstracts, authors, and concepts. Requires the `q` argument.

**Example:**
```sql
SELECT title, publication_year, cited_by_count
FROM arxiv.search(q => 'quantum computing')
LIMIT 5;
```

## Tables

### `papers`
Query papers indexed in arXiv. You can list all papers or filter by specific identifiers:
- `openalex_id` (e.g. `'W2781738013'`)
- `doi` (e.g. `'https://doi.org/10.22331/q-2018-08-06-79'`)
- `arxiv_id` (e.g. `'1706.03762'`)
- `landing_page_url` (e.g. `'http://arxiv.org/abs/cond-mat/0410550'`)
- `publication_year` (e.g. `2024`)

**Examples:**
```sql
-- Retrieve a specific paper by its arXiv ID
SELECT title, publication_year, cited_by_count, pdf_url
FROM arxiv.papers
WHERE arxiv_id = '1706.03762'
LIMIT 1;

-- List preprints from a specific publication year
SELECT title, publication_date, cited_by_count
FROM arxiv.papers
WHERE publication_year = 2025
LIMIT 5;
```
