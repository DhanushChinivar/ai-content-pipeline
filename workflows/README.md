# Workflow exports

The exported n8n workflow JSON lives here (`ai-content-pipeline.json`), added in Phase 9.

## Before committing an export

n8n's export includes credential **references** — IDs and names, not secret values. Still, check every export before it goes into git:

1. Search the JSON for `sk-ant-`, `fc-`, `ntn_`, and your WordPress domain.
2. Replace any credential name that leaks account information.
3. Confirm no node has an API key pasted inline in a header field instead of using a stored credential.

Anything published to a public repo should be treated as permanently public, even if deleted later.

## Importing

n8n → Workflows → Import from File. Credentials do not come with the export — reconnect each one after import (see `../.env.example` for the list).
