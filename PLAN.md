# Plan of Action

Build order is deliberate: **each phase ends with something you can run and see working.** Don't build node 5 before node 1 produces output you trust.

Legend: `[ ]` todo · `[x]` done · 🧠 = a decision to make, not a task to do

---

## Phase 0 — Accounts and keys

- [ ] Google account (Sheets)
- [ ] n8n account, new workflow named **AI Content Pipeline**
- [ ] Firecrawl account + API key
- [ ] Anthropic Console account + API key + **~$5 billing credit** (no free tier; calls 400 without it)
- [ ] Notion account + an **internal integration** (see below)
- [ ] Slack workspace + **incoming webhook** URL for the target channel

### Setting up Notion

1. `notion.so/my-integrations` → **New integration**
2. Name it `n8n Content Pipeline`, pick your workspace, capabilities: **Insert content** + **Read content**
3. Copy the **Internal Integration Secret** (starts `ntn_` or `secret_`)
4. In Notion, create a **database** (not a plain page) called **Content Drafts** with these properties:

   | Property | Type |
   |---|---|
   | `Name` | Title |
   | `Topic` | Text |
   | `Status` | Select — options: `Draft`, `Reviewed` |
   | `Meta Description` | Text |
   | `Tags` | Multi-select |

5. Open the database → **⋯** menu → **Connections** → **Connect to** → your integration

   ⚠️ **This step is the one everyone forgets.** Without it every API call returns `object_not_found` even though the token is valid. The integration can only see pages you explicitly share with it.

6. Copy the **database ID** from the URL:
   `notion.so/workspace/`**`a1b2c3d4e5f6...`**`?v=...` — the 32-character string before the `?`

**Where the secrets actually live:** n8n's credential store, not this repo. `.env.example` documents what's needed so someone else can rebuild it.

---

## Phase 1 — The sheet and the trigger

- [ ] Create sheet **AI Content Pipeline**, tab named **Topics**
- [ ] Row 1 headers, lowercase, no spaces: `topic | status | url | date_added`
- [ ] Freeze row 1 (View → Freeze → 1 row)
- [ ] Add test row: `Best standing desks under $300 | pending | | 2026-08-14`
- [ ] n8n: add **Google Sheets Trigger** node, connect the Google account, select the sheet and the `Topics` tab
- [ ] Run once manually. Confirm the trigger outputs the row as JSON with the four keys.

🧠 **Decision: which trigger event?**

The last node of this pipeline writes `url` and flips `status` to `done` — on the same row that started it.

- **Row Added** — fires only on new rows. Your write-back does not retrigger it.
- **Row Updated** — fires on any cell change. Your write-back *is* a cell change.

Pick "Row Added" unless you have a specific reason not to, and treat `status` as the source of truth for what's been processed rather than as decoration.

**Two things to know now:**
1. n8n **polls** Google Sheets — it does not get a push. Fastest interval is 1 minute. The "5 minutes" in the pitch is realistic, but it is polling latency, not model latency.
2. Header text is the key n8n maps on. `date_added` and `Date Added` are different fields, and a typo surfaces as an empty value three nodes later.

---

## Phase 2 — Research (Firecrawl)

- [ ] Add an **HTTP Request** node after the trigger
- [ ] `POST https://api.firecrawl.dev/v1/search`
- [ ] Header auth: `Authorization: Bearer <FIRECRAWL_API_KEY>` (store as an n8n Header Auth credential, not inline)
- [ ] Body: the topic as the query, limit 5, request markdown in the scrape options
- [ ] Run it. Inspect the output: you want 5 results, each with a title, URL, and a body of markdown.

⚠️ Verify the exact request/response shape against Firecrawl's current API docs before wiring it up — their endpoint versions move faster than tutorials do.

- [ ] Add a **Code** node that flattens the 5 results into one string: `## <title> (<url>)\n<markdown>` per competitor
- [ ] Truncate each competitor's markdown (~4,000 characters is plenty) so one bloated page can't blow up the next step's input

**Why truncate:** you're paying per input token on the next two nodes, and the gap analysis needs *structure and coverage*, not every word.

---

## Phase 3 — Gap analysis (Claude, call 1)

- [ ] Add an **HTTP Request** node: `POST https://api.anthropic.com/v1/messages`
- [ ] Headers: `x-api-key`, `anthropic-version: 2023-06-01`, `content-type: application/json`
- [ ] Model: `claude-opus-5`
- [ ] `max_tokens: 8000`
- [ ] Prompt: [`prompts/01-gap-analysis.md`](prompts/01-gap-analysis.md)
- [ ] Use **structured outputs** so the next node gets reliable JSON instead of prose you have to regex:
      `"output_config": { "format": { "type": "json_schema", "schema": {...} } }`
- [ ] Run it. Read the output yourself. Are the gaps real, or generic filler?

⚠️ **Gotcha:** on `claude-opus-5` thinking is on by default, and `max_tokens` caps thinking **plus** response text together. Set it generously or your JSON truncates mid-object.

**This is the step that makes the whole project defensible.** If the gap analysis is generic, the article is generic and you've built an expensive Jasper. Iterate on this prompt before moving on.

---

## Phase 4 — Writing (Claude, call 2)

- [ ] Second **HTTP Request** node to the same endpoint
- [ ] Model `claude-opus-5`, `max_tokens: 16000`
- [ ] Prompt: [`prompts/02-article-writer.md`](prompts/02-article-writer.md), fed the gap-analysis JSON from Phase 3
- [ ] Structured output with fields: `title`, `meta_title`, `meta_description`, `body_markdown`, `tags`, `gaps_addressed`
- [ ] Run it. Read the whole article. Would you publish it under your name?

🧠 **Decision: one call or two?**
Two calls cost more and take longer than asking for research + article in one shot. Two is still right: the gap analysis is a *checkable intermediate artifact*. When output quality drops you can tell whether the research or the writing failed. One call makes that undebuggable.

**Ask for markdown, not HTML.** The Notion API takes neither — it takes block objects — so markdown is the easier thing to convert from (Phase 5) and the easier thing to eyeball while you're iterating on the prompt.

---

## Phase 5 — Create the Notion draft

The Notion API does **not** accept HTML or markdown. A page's content is an array of typed **block objects**:

```json
{ "object": "block", "type": "heading_2",
  "heading_2": { "rich_text": [{ "type": "text", "text": { "content": "Why height matters" } }] } }
```

So this phase is two nodes, not one.

### 5a — Convert markdown to blocks

- [ ] Add a **Code** node. Split `body_markdown` on newlines and map each line:

  | Markdown | Notion block type |
  |---|---|
  | `## text` | `heading_2` |
  | `### text` | `heading_3` |
  | `- text` | `bulleted_list_item` |
  | anything else non-empty | `paragraph` |

- [ ] Skip blank lines
- [ ] Cap the array at **100 blocks** — Notion rejects more than 100 children in a single create call

A deliberately dumb converter. It handles the four things the writer prompt is told to produce. Don't reach for a markdown library until this actually breaks.

### 5b — Create the page

- [ ] **HTTP Request** node: `POST https://api.notion.com/v1/pages`
- [ ] Headers:
      `Authorization: Bearer <NOTION_TOKEN>` · `Notion-Version: 2022-06-28` · `Content-Type: application/json`
- [ ] Body shape:

```json
{
  "parent": { "database_id": "<NOTION_DATABASE_ID>" },
  "properties": {
    "Name":             { "title": [{ "text": { "content": "<title>" } }] },
    "Topic":            { "rich_text": [{ "text": { "content": "<topic>" } }] },
    "Status":           { "select": { "name": "Draft" } },
    "Meta Description": { "rich_text": [{ "text": { "content": "<meta_description>" } }] },
    "Tags":             { "multi_select": [{ "name": "tag1" }, { "name": "tag2" }] }
  },
  "children": [ ...blocks from 5a... ]
}
```

- [ ] Run it. Open Notion. Are headings, paragraphs, and bullets intact?
- [ ] Capture the returned `url` and `id` — the next two nodes need them

⚠️ **Notion-Version is a required header**, not optional. Omit it and every call fails. `2022-06-28` is the long-stable version; check Notion's current docs if you hit unexpected schema errors.

⚠️ Property names in the JSON must match your database columns **exactly**, including capitalisation.

🧠 **n8n's native Notion node vs HTTP Request?** The native node is faster to set up (OAuth-style credential, form fields instead of raw JSON) but gives you less control over block structure. Try it first; drop to HTTP Request if the block handling fights you. Either way the conversion in 5a still has to happen.

---

## Phase 6 — Write back to the sheet

- [ ] **Google Sheets** node, operation: Update Row
- [ ] Match on the topic (or the row number carried through from the trigger)
- [ ] Set `status = done`, `url = <Notion page url>`
- [ ] Run the full chain end to end. Watch the row update.

**This closes the loop.** Without it you have no idea what's been processed and re-running the workflow duplicates pages.

---

## Phase 7 — Notify Slack

- [ ] **HTTP Request** node: `POST <SLACK_WEBHOOK_URL>`
- [ ] Body: `{"text": "📝 Draft ready: <topic>\n<notion url>"}`
- [ ] Trigger the full pipeline from a fresh sheet row. Confirm the Slack message arrives with a working link.

**Milestone: the pipeline works end to end.** Everything after this is hardening.

---

## Phase 8 — Make it survive contact with reality

- [ ] Set `status = failed` on error instead of leaving the row stuck on `pending`
- [ ] Add an **Error Trigger** workflow that posts failures to Slack — a silent failure is worse than a loud one
- [ ] Handle Firecrawl returning fewer than 5 results (thin niches, obscure queries)
- [ ] Add a retry with backoff on the two Anthropic calls (429s happen)
- [ ] Handle articles that convert to more than 100 blocks — either truncate, or append the remainder with `PATCH /v1/blocks/<page_id>/children`
- [ ] Cap it: don't process more than N rows per run, so one bad paste into the sheet can't fire 200 API calls
- [ ] Confirm the write-back cannot retrigger the workflow (the Phase 1 decision, verified rather than assumed)

---

## Phase 9 — Portfolio polish

- [ ] Export the workflow JSON → `workflows/ai-content-pipeline.json` (**strip credential IDs first**)
- [ ] Screenshot the n8n canvas → README
- [ ] Record a 60-second demo: add row → Slack notification → Notion draft
- [ ] Write up one real before/after: a topic you ran, what the competitors missed, what the pipeline covered instead
- [ ] Note actual cost per article (Firecrawl + Anthropic tokens) — "$0.25 per article vs $50–200 for a writer" is the line that lands in an interview

---

## Open questions

Answer these as you go; they're the interesting parts of the build.

1. **How do you know the article is actually better?** Right now the only judge is you reading it. Is there a checkable signal — word count vs competitors, questions answered, headings covered?
2. **What happens on a topic with no good SERP?** Garbage in, confidently-written garbage out. Should the pipeline refuse rather than write?
3. **Where does the cost actually go?** Measure per node before optimizing. The intuition (writing is the expensive call) may be wrong once competitor markdown is in the input.
4. **Is `status` load-bearing or decorative?** It only matters if something reads it. Does anything?
5. **How much work is the last node really?** Swapping Notion for WordPress or Ghost should be one node. If it turns out to be three, the pipeline is more coupled to its destination than the design claims.
