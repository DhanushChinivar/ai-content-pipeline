# n8n node configurations

Copy-paste reference for each node in the pipeline. Kept here so the exact request bodies survive outside the workflow export, and so the pipeline can be rebuilt by hand.

Node names matter: expressions like `$('Google Sheets Trigger')` reference nodes by their canvas name. Rename a node, update every expression that points at it.

---

## Node 1 — Google Sheets Trigger

| Field | Value |
|---|---|
| Trigger | On row added |
| Credential | Google OAuth (sign-in, no key) |
| Poll Times | Every Minute |
| Document | `AI Content Pipeline` |
| Sheet | the topics tab |

Output: `{ topic, status, url, date_added }`

---

## Node 2 — HTTP Request: Firecrawl search

| Field | Value |
|---|---|
| Method | POST |
| URL | `https://api.firecrawl.dev/v2/search` |
| Authentication | Generic Credential Type → Header Auth |
| Credential name | `Authorization` |
| Credential value | `Bearer fc-...` ← the `Bearer ` prefix is required |
| Send Body | on · JSON · Using JSON · **Expression mode** |

```
{
  "query": "{{ $json.topic }}",
  "limit": 5,
  "scrapeOptions": { "formats": [{ "type": "markdown" }] }
}
```

Output: `data.web[]` with `title`, `url`, `markdown`.

Expect ~1 in 5 results to have no `markdown` — sites like Reddit block scrapers. Not an error.

---

## Node 3 — Code: flatten and truncate

Mode: Run Once for All Items · Language: JavaScript

```javascript
const results = $input.first().json.data.web || [];

const scraped = results.filter(r => r.markdown);

const competitor_markdown = scraped
  .map(r => `## ${r.title}\n(${r.url})\n\n${r.markdown.slice(0, 4000)}`)
  .join('\n\n---\n\n');

return [{
  json: {
    topic: $('Google Sheets Trigger').first().json.topic,
    competitor_count: scraped.length,
    competitor_markdown,
  }
}];
```

Cuts ~139,000 chars to ~16,000 — roughly $0.18 → $0.02 of input tokens per run.

Output: `{ topic, competitor_count, competitor_markdown }`

---

## Node 4 — HTTP Request: Claude gap analysis

| Field | Value |
|---|---|
| Method | POST |
| URL | `https://api.anthropic.com/v1/messages` |
| Authentication | Generic Credential Type → Header Auth |
| Credential name | `x-api-key` |
| Credential value | `sk-ant-...` ← **no** `Bearer` prefix |
| Send Headers | on → `anthropic-version` : `2023-06-01` |
| Send Body | on · JSON · Using JSON · **Expression mode** |

⚠️ A separate credential from Firecrawl's. Different header name, no prefix.

```
{
  "model": "claude-opus-5",
  "max_tokens": 8000,
  "system": "You are an SEO content strategist. You are given the full text of the articles currently ranking on page one for a target keyword. Your job is to find the openings they leave and turn them into an outline for a better article. Be specific and evidence-based. Every gap you name must be traceable to what the competitor articles actually do or don't say. If the competitors genuinely cover the topic well, say so plainly rather than inventing weaknesses.",
  "messages": [
    {
      "role": "user",
      "content": {{ JSON.stringify("TARGET KEYWORD: " + $json.topic + "\n\nCOMPETITOR ARTICLES CURRENTLY RANKING:\n\n" + $json.competitor_markdown + "\n\n---\n\nAnalyse these and produce an outline that would outrank them.") }}
    }
  ],
  "output_config": {
    "format": {
      "type": "json_schema",
      "schema": {
        "type": "object",
        "properties": {
          "competitor_summary": { "type": "string" },
          "common_angle": { "type": "string" },
          "gaps": {
            "type": "array",
            "items": {
              "type": "object",
              "properties": {
                "gap": { "type": "string" },
                "evidence": { "type": "string" }
              },
              "required": ["gap", "evidence"],
              "additionalProperties": false
            }
          },
          "unanswered_questions": { "type": "array", "items": { "type": "string" } },
          "recommended_angle": { "type": "string" },
          "outline": {
            "type": "array",
            "items": {
              "type": "object",
              "properties": {
                "heading": { "type": "string" },
                "covers": { "type": "array", "items": { "type": "string" } },
                "fills_gap": { "type": "string" }
              },
              "required": ["heading", "covers", "fills_gap"],
              "additionalProperties": false
            }
          }
        },
        "required": ["competitor_summary", "common_angle", "gaps", "unanswered_questions", "recommended_angle", "outline"],
        "additionalProperties": false
      }
    }
  }
}
```

### The `JSON.stringify` line

```
"content": {{ JSON.stringify(...) }}
```

No quotes around the expression — `JSON.stringify` supplies them, and escapes the quotes, newlines, and backslashes inside the scraped competitor text. Without it, the first `"` in a competitor article breaks the request body and you get a JSON parse error that points nowhere near the real cause.

### `max_tokens` on `claude-opus-5`

Thinking is on by default and shares the `max_tokens` budget with the response. 8000 leaves room for both. Too low and the structured JSON truncates mid-object.

Output: the analysis JSON lives in `content[0].text` as a **string** — the next node has to `JSON.parse` it.

---

## Node 5 — HTTP Request: Claude article writer

Not built yet. See [`../prompts/02-article-writer.md`](../prompts/02-article-writer.md) for the prompt and schema.

`max_tokens: 16000` — a ~1,200-word body plus thinking.

---

## Node 6 — Code: markdown → Notion blocks

Not built yet. See PLAN.md Phase 5a.

## Node 7 — HTTP Request: create Notion page

Not built yet. See PLAN.md Phase 5b.

## Node 8 — Google Sheets: update row

Not built yet. See PLAN.md Phase 6.

## Node 9 — HTTP Request: Slack webhook

Not built yet. See PLAN.md Phase 7.
