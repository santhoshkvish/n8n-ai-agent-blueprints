# Resume Tailoring Agent (Web App)

A webhook-driven variant of the [Resume Tailoring Agent](../resume-tailoring-agent/README.md),
built to sit behind a custom frontend instead of n8n's built-in chat widget.
Accepts a job description + resume PDF via HTTP POST, and returns a
**structured JSON resume** instead of a Markdown/HTML blob, so the frontend
can render it into editable fields and export to Word/PDF client-side.

**Pattern demonstrated:** the same file-upload → text-extraction →
single-shot rewriting agent as the chat version, but retargeted for
programmatic (non-chat) consumption — webhook trigger, JSON-schema output
contract, CORS-enabled response, and a sanitizer step to guard against
malformed model output.

> **Relationship to the chat version:** this is a separate, parallel
> workflow — not a modification of `resume-tailoring-agent/workflow.json`.
> The chat version keeps using n8n's own file-upload protocol
> (`$json.files[0].binaryKey`) and returns HTML/Markdown for the chat
> widget to render. This version uses raw `multipart/form-data` over a
> Webhook node and returns strict JSON. Keep both if you still want the
> chat-based demo to work.

## Architecture

```
[Webhook: POST /resume-tailor]              [Webhook: OPTIONS /resume-tailor]
      │                                             │
      ▼                                             ▼
[Extract from File: pdf]                [Respond - CORS Preflight (204)]
      │
      ▼
[Resume Generator Agent]  ──uses──▶  [Google Gemini Chat Model]
      │
      ▼
[Sanitize & Parse JSON]   (strips stray ```json fences, JSON.parse, error-wraps)
      │
      ▼
[Respond to Webhook]  → 200 { success, data } or 422 { success:false, error }
```

## Nodes

| Node | Type | Key config |
|---|---|---|
| `Webhook - Resume Tailor` | `webhook` | `POST /resume-tailor`, `responseMode: responseNode`, CORS via `allowedOrigins: *` |
| `Webhook - CORS Preflight` | `webhook` | `OPTIONS /resume-tailor` — handles browser preflight separately from the POST path |
| `Extract from File` | `extractFromFile` | `operation: pdf`, `binaryPropertyName: resume` — **assumes** the frontend's multipart field is named `resume` |
| `Resume Generator Agent` | `agent` | Same ATS-tailoring logic as the chat version; user message pulls the job description from `$('Webhook - Resume Tailor').item.json.body.jobDescription` |
| `Google Gemini Chat Model` | `lmChatGoogleGemini` | Model: `models/gemini-3.6-flash` — verify this string against your AI Studio credential before activating |
| `Sanitize & Parse JSON` | `code` | Strips markdown fences the model may add despite instructions, parses to JSON, wraps failures as `{success:false, error, raw}` instead of breaking silently |
| `Respond to Webhook` | `respondToWebhook` | Returns `200` on success, `422` on parse failure; `Access-Control-Allow-Origin: *` |

## Output contract

On success (`200`):
```json
{
  "success": true,
  "data": {
    "summary": "string",
    "skills": ["string"],
    "experience": [
      { "title": "string", "company": "string", "dates": "string", "bullets": ["string"] }
    ],
    "education": [
      { "degree": "string", "institution": "string", "dates": "string" }
    ],
    "additionalSections": [
      { "heading": "string", "items": ["string"] }
    ],
    "tailoringNotes": {
      "keywordsInjected": ["string"],
      "sectionsRewritten": ["string"],
      "addedSkills": [{ "skill": "string", "justification": "string" }],
      "remainingGaps": ["string"]
    }
  }
}
```

On failure (`422`):
```json
{ "success": false, "error": "Failed to parse agent output as JSON", "message": "...", "raw": "..." }
```

## Credentials required

Same as the chat version — see
[`resume-tailoring-agent/README.md`](../resume-tailoring-agent/README.md#credentials-required):
Google Gemini (PaLM) API, free tier via [aistudio.google.com/apikey](https://aistudio.google.com/apikey)
(1,500 requests/day, no card required).

## Sample data

Uses the same test resumes as the chat version:
- `Alex Turing_resume.pdf` — Senior AI/ML Engineer, 8 yrs, generalist NLP/CV/MLOps background
- `Jordan Patel_resume.pdf` — Staff AI Engineer, 8 yrs, Edge AI / RAG / Computer Vision specialist

These two are useful for testing the JSON contract against genuinely
different resume shapes (different section counts, different seniority
framing) before wiring up the frontend's rendering logic.

## Known differences from the chat version's limitations

- **No more "Markdown mislabeled as HTML" problem** — the sanitizer node
  forces a real parse, so the frontend gets typed fields, not a string to
  guess the format of.
- **PDF-extraction-failure risk still applies unchanged** — a scanned/
  image-only PDF with no text layer will still produce an empty `<resume>`
  block for the agent. Worth adding an IF-node check on `Extract from
  File`'s output length before calling the agent, same TODO as the chat
  version.
- **Single-file assumption still applies** — only the first `resume` field
  is read; no multi-file handling.
- **New failure mode to handle client-side:** unlike the chat version
  (which just shows whatever text comes back), this version can now
  return a `422` with `success:false`. The frontend must handle this case
  explicitly — e.g. show "couldn't parse the tailored resume, try again"
  rather than trying to render `data` that doesn't exist.

## Verify before activating

- `models/gemini-3.6-flash` exact string, and `lmChatGoogleGemini`
  node's `typeVersion`/param names — confirm against your live n8n
  instance, per the same version-drift caveat as the chat workflow.
- Whether your n8n version's Webhook node has a native CORS toggle that
  makes the separate `OPTIONS` preflight node unnecessary.
