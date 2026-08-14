# Prompt 2 — Article writer

**Model:** `claude-opus-5` · **max_tokens:** 16000 · **structured output:** yes

Takes the gap analysis from prompt 1. Returns a publish-ready article plus SEO metadata, as HTML.

---

## System prompt

```
You are a content writer producing a publish-ready article for a company blog.

You are given a research brief: what the currently-ranking articles cover, what
they miss, and an outline built to fill those gaps. Follow the outline. The
sections marked as filling a gap are the reason this article exists — give them
real depth, not a paragraph of acknowledgement.

Write like a knowledgeable person explaining something they actually understand.
Concrete over abstract: specific numbers, named tradeoffs, real scenarios. If
you don't know a fact, write around it rather than inventing it — no fabricated
statistics, prices, study citations, or product specifications.

Avoid the register that makes writing read as machine-generated: no "in today's
fast-paced world", no "it's important to note that", no section that only
restates the section above it, no closing paragraph that summarises what the
reader just read.

Output the body as Markdown, using only these four constructs:
  ## Section heading
  ### Subsection heading
  - Bullet point
  Plain paragraph text

No H1 (the page title is supplied separately), no tables, no code blocks, no
images, no links, no bold or italic markers. Separate every block with a blank
line.
```

## User message template

```
TARGET KEYWORD: {{ $json.topic }}

RESEARCH BRIEF:
{{ JSON.stringify($json.gap_analysis) }}

---

Write the article. Target roughly 1,200 words.
```

## Output schema

```json
{
  "type": "object",
  "properties": {
    "title": {
      "type": "string",
      "description": "The article's H1. Compelling, not keyword-stuffed."
    },
    "meta_title": {
      "type": "string",
      "description": "SEO title tag. Under 60 characters, includes the target keyword."
    },
    "meta_description": {
      "type": "string",
      "description": "SEO meta description. 140-160 characters. Should make someone click."
    },
    "body_markdown": {
      "type": "string",
      "description": "The full article body as Markdown, using only ##, ###, - and plain paragraphs. No H1."
    },
    "tags": {
      "type": "array",
      "description": "3-6 tags for the Notion Tags multi-select.",
      "items": { "type": "string" }
    },
    "gaps_addressed": {
      "type": "array",
      "description": "Which gaps from the brief this draft actually covers, and in which section.",
      "items": { "type": "string" }
    }
  },
  "required": [
    "title", "meta_title", "meta_description",
    "body_markdown", "tags", "gaps_addressed"
  ],
  "additionalProperties": false
}
```

---

## Notes

- **`gaps_addressed` is your quality check.** Compare it against the `gaps` array from prompt 1. If they don't line up, the writing step ignored the research step — and the whole premise of the pipeline is that it doesn't.
- **The four-construct restriction is deliberate.** Notion's API takes neither HTML nor markdown — it takes typed block objects, so a conversion node sits between this prompt and Notion either way. Constraining the model to `##`, `###`, `-`, and paragraphs means that converter stays ~15 lines instead of becoming a markdown parser. If you later need tables or code blocks, widen the prompt *and* the converter together.
- **The "no fabricated facts" instruction is load-bearing**, not boilerplate. A confidently invented price or statistic on a client blog is the failure mode that gets a pipeline like this shut down. Consider it a candidate for a downstream check.
- `max_tokens` is 16000 because thinking shares the budget with a ~1,200-word HTML body on `claude-opus-5`.

## Version log

| Date | Change |
|---|---|
| 2026-08-14 | Initial version |
| 2026-08-14 | Switched `body_html` → `body_markdown`; publishing target moved from WordPress to Notion |
