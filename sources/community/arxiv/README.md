# arXiv Source

Search arXiv papers, preprints, and research metadata.

Under the hood, this source queries the open-access [OpenAlex API](https://openalex.org/), which indexes the entire arXiv preprint repository. This approach allows Coral to fetch and parse paper metadata in native JSON format, bypassing the XML-only limitation of the official arXiv API.

## Rate Limits & Usage

OpenAlex is free and open to the public, but requires a free API key to get access to higher rate limits (up to 100 requests per second) and faster response times.
- **Rate limit:** 100 requests per second with an API key.
- Get a free key by visiting the [OpenAlex Settings Page](https://openalex.org/settings/api).

## Setup

Configure the `OPENALEX_API_KEY` credential in your environment:

```bash
export OPENALEX_API_KEY="your_free_api_key"
```

Add the source directly using the CLI:

```bash
coral source add --file sources/community/arxiv/manifest.yaml
```

## Functions

### `search(q)`
Perform a full-text search across arXiv preprint titles, abstracts, and full text. Requires the `q` argument.

**Example:**
```sql
SELECT title, publication_year, cited_by_count, best_oa_arxiv_id
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

#### Output Columns of Interest
- `best_oa_arxiv_id`: The arXiv identifier of the work derived from the best open access location (e.g. `'1706.03762'`). Note that `best_oa_arxiv_id` may be null or contain a non-arXiv URL if the best open access location for a paper is a publisher page, journal, or DOI link rather than arxiv.org.
- `best_oa_landing_page_url` / `best_oa_pdf_url`: The best Open Access URLs for the paper (which may point to a publisher page or arXiv).
- `locations`: JSON array of all hosted copies/repositories. Inspect this to find explicit arXiv links or other versions.

**Examples:**
```sql
-- Retrieve a specific paper by its arXiv ID
SELECT title, publication_year, cited_by_count, best_oa_pdf_url, best_oa_arxiv_id
FROM arxiv.papers
WHERE arxiv_id = '1706.03762'
LIMIT 1;

-- List preprints from a specific publication year
SELECT title, publication_date, cited_by_count, best_oa_landing_page_url
FROM arxiv.papers
WHERE publication_year = 2025
LIMIT 5;
```
