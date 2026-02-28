# API Reference for Paper Citation

This document contains detailed API specifications for academic paper retrieval services.

---

## Semantic Scholar API

### Paper Search Endpoint

**URL:** `https://api.semanticscholar.org/graph/v1/paper/search`

**Method:** GET

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| query | string | Yes | Search query string (URL encoded) |
| limit | integer | No | Max results (default: 10, max: 100) |
| offset | integer | No | Pagination offset |
| fields | string | No | Comma-separated list of fields to return |

**Available Fields:**

```
paperId, url, title, abstract, venue, year, referenceCount, citationCount,
influentialCitationCount, isOpenAccess, openAccessPdf, fieldsOfStudy,
publicationTypes, publicationDate, journal, citationStyles, authors,
externalIds, tldr
```

**Example Request:**
```bash
curl -s "https://api.semanticscholar.org/graph/v1/paper/search?query=virtual%20reality%20museum&limit=10&fields=title,authors,year,citationCount,venue,externalIds,abstract"
```

**Example Response:**
```json
{
  "total": 1234,
  "offset": 0,
  "next": 10,
  "data": [
    {
      "paperId": "abc123",
      "title": "Virtual Reality in Museums: A Study",
      "authors": [
        {"authorId": "auth1", "name": "John Smith"},
        {"authorId": "auth2", "name": "Jane Doe"}
      ],
      "year": 2023,
      "citationCount": 45,
      "venue": "Museum Management",
      "externalIds": {
        "DOI": "10.1234/example.2023",
        "MAG": "12345678"
      },
      "abstract": "This paper explores..."
    }
  ]
}
```

### Paper Details Endpoint

**URL:** `https://api.semanticscholar.org/graph/v1/paper/{paper_id}`

**Method:** GET

**Example:**
```bash
curl -s "https://api.semanticscholar.org/graph/v1/paper/abc123?fields=title,authors,year,citationCount,references,citations"
```

### Batch Paper Details

**URL:** `https://api.semanticscholar.org/graph/v1/paper/batch`

**Method:** POST

**Request Body:**
```json
{
  "ids": ["abc123", "def456"]
}
```

---

## OpenAlex API

### Works Search Endpoint

**URL:** `https://api.openalex.org/works`

**Method:** GET

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| search | string | General search query |
| filter | string | Filter conditions |
| sort | string | Sort field |
| per_page | integer | Results per page (max: 200) |
| page | integer | Page number |
| select | string | Fields to return |

**Example Request:**
```bash
curl -s "https://api.openalex.org/works?search=virtual%20reality%20museum&per_page=10&select=id,doi,title,authorships,publication_year,cited_by_count"
```

**Example Response:**
```json
{
  "meta": {
    "count": 1234,
    "db_response_time_ms": 45,
    "page": 1,
    "per_page": 10
  },
  "results": [
    {
      "id": "https://openalex.org/W123456789",
      "doi": "https://doi.org/10.1234/example",
      "title": "Virtual Reality in Museums: A Study",
      "authorships": [
        {
          "author": {"display_name": "John Smith"},
          "institutions": [{"display_name": "University Name"}]
        }
      ],
      "publication_year": 2023,
      "cited_by_count": 45,
      "primary_location": {
        "source": {
          "display_name": "Museum Management"
        }
      },
      "type": "article"
    }
  ]
}
```

### Work Details Endpoint

**URL:** `https://api.openalex.org/works/{work_id}`

**Method:** GET

**Example:**
```bash
curl -s "https://api.openalex.org/works/W123456789"
```

### Filter Options

Common filters for OpenAlex:

```
?filter=publication_year:2020-2023
?filter=type:article
?filter=is_oa:true
?filter=concepts.id:C123456789
```

---

## CrossRef API

### Works Endpoint

**URL:** `https://api.crossref.org/works/{doi}`

**Method:** GET

**Example Request:**
```bash
curl -s "https://api.crossref.org/works/10.1145/1234567"
```

**BibTeX Content Negotiation:**
```bash
curl -s -LH "Accept: application/x-bibtex" "https://doi.org/10.1145/1234567"
```

**Example Response (JSON):**
```json
{
  "status": "ok",
  "message-type": "work",
  "message": {
    "DOI": "10.1145/1234567",
    "title": ["Paper Title"],
    "author": [
      {"given": "John", "family": "Smith"},
      {"given": "Jane", "family": "Doe"}
    ],
    "published-print": {"date-parts": [[2023, 5, 15]]},
    "container-title": ["Conference Name"],
    "type": "proceedings-article"
  }
}
```

### Query Parameters

| Parameter | Description |
|-----------|-------------|
| query | Search query |
| filter | Filter results |
| sort | Sort order |
| order | asc or desc |
| rows | Number of results |
| offset | Pagination offset |

---

## DOI.org

### Content Negotiation

DOI.org supports content negotiation for different formats:

**BibTeX:**
```bash
curl -s -LH "Accept: application/x-bibtex" "https://doi.org/10.xxxx/xxxxx"
```

**JSON (CrossRef format):**
```bash
curl -s -LH "Accept: application/vnd.citationstyles.csl+json" "https://doi.org/10.xxxx/xxxxx"
```

**RDF:**
```bash
curl -s -LH "Accept: application/rdf+xml" "https://doi.org/10.xxxx/xxxxx"
```

---

## Error Handling

### Semantic Scholar

| Status | Meaning |
|--------|---------|
| 200 | Success |
| 400 | Bad request |
| 404 | Paper not found |
| 429 | Rate limit exceeded |
| 500 | Server error |

### OpenAlex

| Status | Meaning |
|--------|---------|
| 200 | Success |
| 400 | Bad request |
| 403 | Forbidden (rate limit) |
| 404 | Not found |

### CrossRef

| Status | Meaning |
|--------|---------|
| 200 | Success |
| 404 | DOI not found |
| 503 | Service unavailable |

---

## Rate Limits

| Service | Limit | Notes |
|---------|-------|-------|
| Semantic Scholar | 100 req/5min | No API key required |
| OpenAlex | 100000 req/day | Be polite, no key needed |
| CrossRef | 50 req/sec | Use polite pool with mailto |
| DOI.org | No limit | But be respectful |

**Polite User-Agent for CrossRef:**
```bash
curl -s -A "MyTool/1.0 (mailto:myemail@example.com)" "https://api.crossref.org/..."
```

---

## Data Mapping

### Normalized Paper Schema

When merging results from multiple APIs, use this common schema:

```json
{
  "id": "unique identifier",
  "title": "paper title",
  "authors": ["Author 1", "Author 2"],
  "year": 2023,
  "venue": "Journal or Conference Name",
  "citation_count": 45,
  "doi": "10.xxxx/xxxxx",
  "abstract": "paper abstract",
  "url": "https://...",
  "source_api": "semanticscholar|openalex"
}
```

### Author Name Normalization

Semantic Scholar: `{"name": "John Smith"}`
OpenAlex: `{"author": {"display_name": "John Smith"}}`
CrossRef: `{"given": "John", "family": "Smith"}`

**Normalized:** `"John Smith"`

---

## Useful Tips

1. **URL Encoding:** Always encode query parameters
2. **Pagination:** Use offset/limit or page/per_page consistently
3. **Field Selection:** Use `fields` or `select` to reduce response size
4. **Caching:** Cache venue rankings locally to reduce web searches
5. **Fallbacks:** If one API fails, try the other
6. **DOI Resolution:** Try CrossRef first, then DOI.org as fallback
