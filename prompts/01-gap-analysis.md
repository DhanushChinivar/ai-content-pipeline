# Prompt 1 — Competitor gap analysis

**Model:** `claude-opus-5` · **max_tokens:** 8000 · **structured output:** yes

Takes the scraped markdown of the top 5 ranking articles. Returns a JSON outline built around what those articles *fail* to cover.

---

## System prompt

```
You are an SEO content strategist. You are given the full text of the articles
currently ranking on page one for a target keyword. Your job is to find the
openings they leave and turn them into an outline for a better article.

Be specific and evidence-based. Every gap you name must be traceable to what
the competitor articles actually do or don't say — not to what articles on this
topic generally tend to miss. If the competitors genuinely cover the topic well,
say so plainly rather than inventing weaknesses.
```

## User message template

```
TARGET KEYWORD: {{ $json.topic }}

COMPETITOR ARTICLES CURRENTLY RANKING:

{{ $json.competitor_markdown }}

---

Analyse these and produce an outline that would outrank them.
```

## Output schema

```json
{
  "type": "object",
  "properties": {
    "competitor_summary": {
      "type": "string",
      "description": "2-3 sentences on what the ranking articles collectively cover and how they're structured."
    },
    "common_angle": {
      "type": "string",
      "description": "The framing nearly all of them share. This is what makes them interchangeable."
    },
    "gaps": {
      "type": "array",
      "description": "Specific things the competitors miss. 3-6 items.",
      "items": {
        "type": "object",
        "properties": {
          "gap": { "type": "string" },
          "evidence": {
            "type": "string",
            "description": "Why you believe this is a gap, referencing the competitor content."
          }
        },
        "required": ["gap", "evidence"],
        "additionalProperties": false
      }
    },
    "unanswered_questions": {
      "type": "array",
      "description": "Questions a reader searching this term would have that no ranking article answers.",
      "items": { "type": "string" }
    },
    "recommended_angle": {
      "type": "string",
      "description": "The single positioning this article should take to be distinct."
    },
    "outline": {
      "type": "array",
      "description": "H2 sections in order, each with the points it must cover.",
      "items": {
        "type": "object",
        "properties": {
          "heading": { "type": "string" },
          "covers": { "type": "array", "items": { "type": "string" } },
          "fills_gap": {
            "type": "string",
            "description": "Which gap or unanswered question this section addresses. Empty if it's table-stakes coverage."
          }
        },
        "required": ["heading", "covers", "fills_gap"],
        "additionalProperties": false
      }
    }
  },
  "required": [
    "competitor_summary", "common_angle", "gaps",
    "unanswered_questions", "recommended_angle", "outline"
  ],
  "additionalProperties": false
}
```

---

## Notes

- **`evidence` is the anti-hallucination field.** Without it the model produces plausible-sounding gaps that aren't grounded in the scraped text. Read it when you're evaluating output quality — if the evidence is vague, the gap is invented.
- **`fills_gap` keeps the outline honest.** If most sections have it empty, the pipeline is about to write the same article the competitors already wrote.
- Thinking is on by default on `claude-opus-5` and shares the `max_tokens` budget with the response. 8000 leaves room for both.

## Version log

| Date | Change |
|---|---|
| 2026-08-14 | Initial version |
