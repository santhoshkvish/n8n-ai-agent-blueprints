# Contributing a New Workflow

This repo scales by every workflow following the **same shape**. That means
anyone (including future-you) can open `workflows/` and immediately know
where to look, without re-reading each workflow's internal structure from
scratch.

## Checklist for adding a workflow

1. **Copy the template**
   ```bash
   cp -r _template workflows/<your-workflow-slug>
   ```
   Use a lowercase, hyphenated slug (`invoice-parser-agent`, not `Invoice Parser`).

2. **Build the workflow in n8n**, then export it (**Workflows → Download**)
   and save it as `workflows/<slug>/workflow.json`.

3. **No secrets in the export.** Before committing:
   - Search the exported JSON for anything that looks like a key: `grep -Ei "bearer|api[_-]?key|secret|token" workflow.json`
   - Every credential must be referenced via n8n's **Credentials** system
     (shows up in the JSON as a `credentials` block with an `id`/`name`,
     never a raw key inside `parameters`).
   - See [`docs/security.md`](docs/security.md).

4. **Write `workflows/<slug>/README.md`** using `_template/README.template.md`
   as the skeleton. At minimum, document:
   - What the workflow does (1-2 sentences)
   - A plain-language architecture diagram (trigger → steps → output)
   - Every node, its type, and the key parameters/expressions it uses
   - Every credential required and where to get the API key
   - Any prompts used (system messages, agent instructions) in full
   - Known limitations / things you'd change for production

5. **Add sample/test data** if the workflow needs input to demo (PDFs,
   sample payloads, etc.) under `workflows/<slug>/sample-data/`.

6. **Add a row to the root `README.md`** workflow index table.

## Naming conventions

- Node names in n8n: `<Role> - <Purpose>`, e.g. `Agent - Invoice Classifier`,
  matching the existing pattern (`Agent - Tech Trends Extractor`).
- Workflow file: `workflow.json` — always this exact name inside each
  workflow's folder, so tooling/scripts can find it predictably.
- Folder slug matches the workflow's display name, lowercased and hyphenated.

## Model/provider choices

Default to OpenRouter unless there's a specific reason not to (keeps
credential setup consistent across the repo). If a workflow needs a
different provider, say why in its README.
