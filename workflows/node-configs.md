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

### `max_tokens` on `claude-opus-5` — now 16000, was 8000

Thinking is on by default and shares the `max_tokens` budget with the response.

**8000 was not enough and it failed intermittently**, which is the worst kind of failure. A run on 2026-08-14 truncated the JSON at ~19,000 characters and surfaced in the *next* node as:

```
Unterminated string in JSON at position 19011 (line 1 column 19012)
```

That message points at `JSON.parse` in the Code node, which is not where the problem is. The response simply stopped mid-string because thinking plus output hit the ceiling. Raised to **16000**.

Both Code nodes that read a Claude response now check `stop_reason` first and throw a message that names the real cause:

```javascript
if (res.stop_reason === 'max_tokens') {
  throw new Error('Claude hit max_tokens and the JSON is cut off. Raise max_tokens on the previous node.');
}
```

If you see that error, raise the ceiling on the node before it — don't touch the parsing.

Output: the analysis JSON arrives as a **string** inside a `text` block, so the next node has to `JSON.parse` it.

⚠️ **It is not `content[0]`.** Thinking is on by default, so `content[0]` is a `thinking` block and the text block sits at `content[1]`. `content[0].text` is `undefined`, and `JSON.parse(undefined)` fails with the unhelpful `"undefined" is not valid JSON`. Find the block by type instead — the block order is not part of the API contract:

```javascript
const content = $input.first().json.content || [];
const textBlock = content.find(b => b.type === 'text');
```

Both Code nodes that read a Claude response (`Parse Gap Analysis`, `Markdown To Notion Blocks`) do this.

---

## Node 5 — HTTP Request: Claude article writer

On the canvas. Prompt and schema: [`../prompts/02-article-writer.md`](../prompts/02-article-writer.md).

`max_tokens: 16000` — a ~1,200-word body plus thinking.

---

## Node 6 — Code: markdown → Notion blocks

On the canvas. Exact body in [`ai-content-pipeline.json`](ai-content-pipeline.json) under `Markdown To Notion Blocks`. Caps the array at 100 blocks.

## Node 7 — HTTP Request: create Notion page

On the canvas. `POST https://api.notion.com/v1/pages`, header `Notion-Version: 2022-06-28`.

The live node has the real `database_id`; this repo keeps the `PASTE_YOUR_NOTION_DATABASE_ID_HERE` placeholder on purpose, since the repo is public.

Notion setup, as actually built:

- Connection type is **Access token**, not OAuth. OAuth is for apps acting on behalf of many signed-in users; this node authenticates as itself with one static header.
- The `Content Drafts` database property names must match the request body exactly: `Name` (title), `Topic` (text), `Status` (**Select**, not Notion's separate "Status" property type — the body sends `{ select: { name: 'Draft' } }`), `Meta Description` (text), `Tags` (multi-select).
- Select and multi-select options don't need pre-creating; the API adds unknown option names on write.
- The database's **⋯ → Connections** must list the integration. Creating the connection grants nothing on its own — without this the API returns `object_not_found` for a database you can see in the sidebar.

## Node 8 — Set: prepare sheet update

An **Edit Fields (Set)** node between `Create Notion Page` and `Update Sheet Row`. The Notion response is a page object — it carries no `topic`, so this node rebuilds the three fields the sheet needs:

| Name | Type | Value |
|---|---|---|
| `topic` | String | `{{ $('Markdown To Notion Blocks').first().json.topic }}` |
| `status` | String | `done` |
| `url` | String | `{{ $json.url }}` |

**Include Other Input Fields** off, so nothing from the Notion response leaks into the sheet write.

⚠️ Watch for a stray space after the `=` expression marker. `"= {{ $json.url }}"` is a *different* value from `"={{ $json.url }}"` — the first writes the URL with a leading space into the cell. The editor hides the `=`, so the space is nearly invisible.

## Node 9 — Google Sheets: update row

On the canvas and working. Document by ID, Sheet **From list → `Untitled`**, operation Update Row, match on `topic`, writes `status` = `done` and `url` = `{{ $json.url }}` (Notion's create-page response includes the page `url`).

Needs its own credential: the trigger's is type `googleSheetsTriggerOAuth2Api`, a regular Google Sheets node needs `googleSheetsOAuth2Api` — **a different type, so it cannot be reused.** Connect Google a second time on this node.

⚠️ **Sheet "By ID" wants the bare gid, not `gid=0`.** The node had the literal string `gid=0`, which silently resolves to nothing: the node shows *"No columns found in Google Sheets"* and the whole field mapping stays empty. Either use `0`, or switch the selector to **From list** and pick the tab — the list is also the quickest way to confirm the credential can actually read the document.

⚠️ **The tab is named `Untitled`, not `Topics`.** Step 1 of Phase 1 says to name it `Topics`; that never happened. The name doesn't matter to the pipeline, but don't go looking for a `Topics` tab that isn't there.

⚠️ **A match that finds nothing is silent.** This node ran green, returned zero items and wrote nothing, with no error message anywhere — for a whole afternoon. The cause was not the node at all: the trigger had **pinned data** from `Copy to editor`, holding a topic that had since been edited out of the sheet. The node dutifully searched for a row that no longer existed.

Two lessons worth more than the fix:

- **A Google Sheets update whose match value hits no row does not error.** It returns nothing and the execution is marked successful. Treat "no output data" from this node as *"no row matched"* and go compare the match value against the sheet's actual contents first.
- **The trigger's output panel is not the sheet.** It is a snapshot taken when the run started, so it shows `status: pending` and an empty `url` on every successful run, forever. Only open the spreadsheet to confirm a write.

Pinned data is easy to miss: n8n greys out the node's parameters and puts a small pin badge on the node. Select the node and press `p` to unpin.

⚠️ **Only map the columns you intend to overwrite.** n8n pre-fills *every* column in "Values to Update", so `date_added` (blank) and `row_number` (`0`) were both mapped and would have wiped those cells on every run. Delete the rows you don't want written.

## ~~Node 10 — HTTP Request: Slack webhook~~ — cut 2026-08-14

Removed from the workflow. It only announced a draft that the Notion database and the sheet's `url` column already record. See Phase 7 in [`../PLAN.md`](../PLAN.md) for the reasoning and the condition under which it would be worth adding back.

---

## Credentials: three Header Auth credentials, not one

Four nodes authenticate with the generic **Header Auth** type, and they need **three separate credentials** because the header name and prefix differ per service:

| Credential | Header name | Value | Used by |
|---|---|---|---|
| Firecrawl | `Authorization` | `Bearer fc-...` | Firecrawl Search |
| Anthropic | `x-api-key` | `sk-ant-...` (no prefix) | Claude Gap Analysis, Claude Write Article |
| Notion | `Authorization` | `Bearer ntn_...` | Create Notion Page |

⚠️ **n8n auto-selects the only existing Header Auth credential when you add a node.** That is how this build broke once: editing the single `Header Auth account` from Firecrawl's `Authorization` to Anthropic's `x-api-key` silently repointed the Firecrawl node too, and its next run 401s. One credential per service — create them separately and check which one each node has selected.
