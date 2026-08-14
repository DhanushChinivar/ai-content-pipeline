# Plan of Action

Build order is deliberate: **each phase ends with something you can run and see working.** Don't build node 5 before node 1 produces output you trust.

Legend: `[ ]` todo · `[x]` done · 🧠 = a decision to make, not a task to do

---

## Where the build actually is (2026-08-14)

Live workflow: n8n Cloud instance `dash280`, workflow **AI Content Pipeline** — 10 nodes on the canvas and connected, **published** as `v1 live trigger`. The Google Sheets trigger polls every minute for added rows. (Slack was cut; see Phase 7.)

Note that `rowAdded` only fires for rows added *after* the workflow was published. Rows already sitting at `pending` are invisible to it — to reprocess one, delete it and paste it back.

**End to end and verified — execution #16, 2026-08-14, 4m 16s.** A row pasted into the sheet triggered the workflow on its own, no manual start: trigger → Firecrawl search → flatten → Claude gap analysis → parse → Claude write article → markdown→Notion blocks → Notion page → prepare → sheet write-back. The row flipped `pending` → `done` with the Notion URL in column C, and the draft (*The Best Mechanical Keyboards Under $150 in 2026*) renders with headings, paragraphs, bullets and all five properties intact.

The gap analysis output is genuinely evidence-based, not filler — it names the ranking sites, their actual prices, and a date/price-band mismatch on one of them. That was the open question in Phase 3, and the answer is yes.

What remains is hardening (Phase 8) and polish (Phase 9), not correctness.

### Fixed on 2026-08-14

- Firecrawl Header Auth credential recreated; `Firecrawl Search` no longer 401s.
- **`Update Sheet Row` was silently broken.** Sheet "By ID" held the literal string `gid=0` instead of `0`, so n8n reported *"No columns found in Google Sheets"* and every field mapping was empty. Now set via **From list**. Also discovered the tab is named `Untitled`, not `Topics` — Phase 1 step 2 never happened. And `date_added` (blank) plus `row_number` (`0`) were mapped for writing and would have wiped those cells each run; both removed.
- **Gap analysis `max_tokens` raised 8000 → 16000.** At 8000 the structured JSON truncated at ~19,000 chars and surfaced one node later as `Unterminated string in JSON at position 19011` — an error that points at the parser rather than the cause. Both Code nodes now check `stop_reason === 'max_tokens'` first and say so plainly.
- **`Update Sheet Row` wrote nothing for hours, with no error.** The trigger had **pinned data** left behind by n8n's *"Copy to editor"*, holding the topic `Best standing desks under $300` — a row that had since been edited out of the sheet. The update node searched for a row that no longer existed, matched nothing, and returned zero items. A Google Sheets update that matches nothing does not raise an error and the execution is still marked successful, which is why this masqueraded as a credential problem. Trigger unpinned; see the warning under Node 9 in `workflows/node-configs.md`.
- **The trigger's output panel is not the sheet.** It is a snapshot from the moment the run started, so it shows `status: pending` and an empty `url` after *every* successful run. Confirm writes by opening the spreadsheet.
- **`Prepare Sheet Update` added** — a Set node between `Create Notion Page` and `Update Sheet Row`. The Notion response carries no `topic`, so this rebuilds `topic`/`status`/`url` explicitly rather than relying on a cross-node expression at the write.
- **`content[0].text` was wrong in both Code nodes.** Thinking is on, so `content[0]` is a `thinking` block and the text sits at `content[1]`; `JSON.parse(undefined)` failed with `"undefined" is not valid JSON`. Both now find the block by `type === 'text'`. See `workflows/node-configs.md`.

---

## Phase 0 — Accounts and keys

- [x] Google account (Sheets)
- [x] n8n account, new workflow named **AI Content Pipeline**
- [x] Firecrawl account + API key
- [x] Anthropic Console account + API key + **~$5 billing credit** (no free tier; calls 400 without it)
- [x] Notion account + an **internal integration** (see below)

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

- [x] Create sheet **AI Content Pipeline**, tab named **Topics**
- [x] Row 1 headers, lowercase, no spaces: `topic | status | url | date_added`
- [x] Freeze row 1 (View → Freeze → 1 row)
- [x] Add test row: `Best standing desks under $300 | pending | | 2026-08-14`
- [x] n8n: add **Google Sheets Trigger** node, connect the Google account, select the sheet and the `Topics` tab
- [x] Run once manually. Confirm the trigger outputs the row as JSON with the four keys.

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

- [x] Add an **HTTP Request** node after the trigger
- [x] `POST https://api.firecrawl.dev/v2/search`
- [x] Header auth: `Authorization: Bearer <FIRECRAWL_API_KEY>` (store as an n8n Header Auth credential, not inline)
- [x] Body:

```json
{
  "query": "{{ $json.topic }}",
  "limit": 5,
  "scrapeOptions": { "formats": [{ "type": "markdown" }] }
}
```

- [x] Run it. Inspect the output: 5 results under `data.web`, each with `title`, `url`, and `markdown`.

⚠️ **v2, not v1** — verified against Firecrawl's docs 2026-08-14. Their endpoint versions move faster than tutorials do; re-check if you get a 404.

⚠️ Without `scrapeOptions`, `markdown` comes back `null` — you get search results but no article text, and the gap analysis has nothing to read.

- [x] Add a **Code** node that flattens `data.web` into one string: `## <title> (<url>)\n<markdown>` per competitor
- [x] Truncate each competitor's markdown (~4,000 characters is plenty) so one bloated page can't blow up the next step's input

**Why truncate:** you're paying per input token on the next two nodes, and the gap analysis needs *structure and coverage*, not every word.

---

## Phase 3 — Gap analysis (Claude, call 1)

- [x] Add an **HTTP Request** node: `POST https://api.anthropic.com/v1/messages`
- [x] Headers: `x-api-key`, `anthropic-version: 2023-06-01`, `content-type: application/json`
- [x] Model: `claude-opus-5`
- [x] `max_tokens: 16000` (was 8000 — truncated the JSON mid-string on a real run)
- [x] Prompt: [`prompts/01-gap-analysis.md`](prompts/01-gap-analysis.md)
- [x] Use **structured outputs** so the next node gets reliable JSON instead of prose you have to regex:
      `"output_config": { "format": { "type": "json_schema", "schema": {...} } }`
- [x] Run it. Read the output yourself. Are the gaps real, or generic filler?

⚠️ **Gotcha:** on `claude-opus-5` thinking is on by default, and `max_tokens` caps thinking **plus** response text together. Set it generously or your JSON truncates mid-object.

**This is the step that makes the whole project defensible.** If the gap analysis is generic, the article is generic and you've built an expensive Jasper. Iterate on this prompt before moving on.

---

## Phase 4 — Writing (Claude, call 2)

- [x] Second **HTTP Request** node to the same endpoint
- [x] Model `claude-opus-5`, `max_tokens: 16000`
- [x] Prompt: [`prompts/02-article-writer.md`](prompts/02-article-writer.md), fed the gap-analysis JSON from Phase 3
- [x] Structured output with fields: `title`, `meta_title`, `meta_description`, `body_markdown`, `tags`, `gaps_addressed`
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

- [x] Add a **Code** node. Split `body_markdown` on newlines and map each line:

  | Markdown | Notion block type |
  |---|---|
  | `## text` | `heading_2` |
  | `### text` | `heading_3` |
  | `- text` | `bulleted_list_item` |
  | anything else non-empty | `paragraph` |

- [x] Skip blank lines
- [x] Cap the array at **100 blocks** — Notion rejects more than 100 children in a single create call

A deliberately dumb converter. It handles the four things the writer prompt is told to produce. Don't reach for a markdown library until this actually breaks.

### 5b — Create the page

- [x] **HTTP Request** node: `POST https://api.notion.com/v1/pages`
- [x] Headers:
      `Authorization: Bearer <NOTION_TOKEN>` · `Notion-Version: 2022-06-28` · `Content-Type: application/json`
- [x] Body shape:

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

- [x] Run it. Open Notion. Are headings, paragraphs, and bullets intact? **Yes** — verified 2026-08-14 on *The Best Mechanical Keyboards Under $150 in 2026*: H2/H3 headings, paragraphs and bulleted lists all render, and all five properties (Name, Topic, Status=Draft, Meta Description, six Tags) populated.
- [x] Capture the returned `url` and `id` — the next two nodes need them

⚠️ **Notion-Version is a required header**, not optional. Omit it and every call fails. `2022-06-28` is the long-stable version; check Notion's current docs if you hit unexpected schema errors.

⚠️ Property names in the JSON must match your database columns **exactly**, including capitalisation.

🧠 **n8n's native Notion node vs HTTP Request?** The native node is faster to set up (OAuth-style credential, form fields instead of raw JSON) but gives you less control over block structure. Try it first; drop to HTTP Request if the block handling fights you. Either way the conversion in 5a still has to happen.

---

## Phase 6 — Write back to the sheet

- [x] **Google Sheets** node, operation: Update Row
- [x] Match on the topic (or the row number carried through from the trigger)
- [x] Set `status = done`, `url = <Notion page url>`
- [x] Run the full chain end to end. Watch the row update. **Done 2026-08-14, execution #16** — triggered by a genuinely new row, no manual start, succeeded in 4m 16s. Row 3 flipped `pending` → `done` with the Notion URL written to column C.

**This closes the loop.** Without it you have no idea what's been processed and re-running the workflow duplicates pages.

---

## Phase 7 — ~~Notify Slack~~ (cut 2026-08-14)

**Cut deliberately.** The pipeline is complete without it: the Notion page exists and the sheet row reads `done` with a link. Slack was notification only — it produced nothing. For a single operator adding topics and checking Notion in the same sitting, it announces something you already know.

The node is deleted from the live workflow and from `ai-content-pipeline.json`.

Worth revisiting only if this ever runs at volume, and then in a smarter shape than an unconditional ping per article — e.g. notify only when `competitor_count` came back low, meaning the gap analysis worked from thin input. `Flatten Competitors` already computes that number.

- [x] Trigger the full pipeline from a fresh sheet row.

**Milestone: the pipeline works end to end.** Everything after this is hardening.

---

## Phase 8 — Make it survive contact with reality

- [ ] Set `status = failed` on error instead of leaving the row stuck on `pending`
- [ ] Add an **Error Trigger** workflow so failures surface somewhere — a silent failure is worse than a loud one. With Slack cut, the cheapest signal is setting `status = failed` on the row (above) and checking the n8n executions list.
- [ ] Handle Firecrawl returning fewer than 5 results (thin niches, obscure queries)
- [ ] Add a retry with backoff on the two Anthropic calls (429s happen)
- [ ] Handle articles that convert to more than 100 blocks — either truncate, or append the remainder with `PATCH /v1/blocks/<page_id>/children`
- [ ] Cap it: don't process more than N rows per run, so one bad paste into the sheet can't fire 200 API calls
- [x] Confirm the write-back cannot retrigger the workflow — the trigger fires on `rowAdded`, and the write-back is an *update* to an existing row, never an append. Do not switch `Update Sheet Row` to "Append or Update": on a failed match it would append, which the trigger would then pick up as a new row, and the pipeline would eat itself.

---

## Phase 9 — Portfolio polish

- [x] Export the workflow JSON → `workflows/ai-content-pipeline.json` (credential blocks stripped, Notion database ID replaced with the placeholder)
- [ ] Screenshot the n8n canvas → README
- [ ] Record a 60-second demo: add row → Notion draft appears → sheet row flips to `done` with the link
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
