# Credential Handling Policy

## Why this doc exists

An earlier version of the Tech Trends Extractor workflow had a live Tavily
API key hardcoded in plaintext inside the exported `workflow.json` (in an
HTTP Request node's Authorization header). Any n8n "Download" export embeds
whatever is typed directly into node parameters — including secrets — as
plain text in the JSON. If that file is shared, committed, or zipped up,
the key goes with it.

**Rule: no API key, token, or secret is ever typed directly into a node
parameter.** Always use n8n's **Credentials** system.

## How to do it right

### For nodes with built-in credential types
(e.g. OpenRouter Chat Model, most first-party integrations)

Go to **Credentials → New**, pick the matching credential type, paste the
key there. Select it from the node's **Credential** dropdown. The exported
JSON will only ever contain a credential `id` and `name` reference — never
the key itself.

### For generic HTTP Request nodes calling third-party REST APIs
(e.g. Tavily, or any API without a dedicated n8n node)

1. **Credentials → New → Header Auth** (or **Generic Credential Type**
   depending on your n8n version).
2. Set the header name (e.g. `Authorization`) and value (e.g. `Bearer <key>`)
   there.
3. In the HTTP Request node, set **Authentication** →
   **Generic Credential Type** → **Header Auth**, and select your credential.

This is the pattern used to fix `workflows/tech-trends-extractor/workflow.json`
— check that file's HTTP Request node as a reference example.

## Before committing or sharing any workflow export

Run a quick grep for common secret patterns:

```bash
grep -Ei "bearer|api[_-]?key|secret|token|sk-|tvly-" workflow.json
```

If anything other than a credential `id`/`name` reference shows up, stop and
fix it before the file leaves your machine.

## If a key does leak

1. Revoke/rotate it immediately at the provider (don't just remove it from
   the file — assume it's already been seen).
2. Re-issue and store the new key only via the Credentials system.
3. Re-export the workflow and re-verify with the grep check above.
