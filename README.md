# n8n AI Agent Blueprints

A growing library of documented, reusable **n8n AI agent workflows** — reference
implementations of common agentic patterns (tool-calling, RAG-style retrieval,
multi-agent pipelines, document processing) built on n8n's LangChain nodes.

Each workflow lives in its own folder under `workflows/`, is self-contained,
and follows the same documentation structure so you can drop in a new one
without re-deriving conventions. See [`CONTRIBUTING.md`](CONTRIBUTING.md)
for how to add one.

---

## Workflow Index

| Workflow | Pattern demonstrated | LLM Provider | External APIs |
|---|---|---|---|
| [Tech Trends Extractor](workflows/tech-trends-extractor/README.md) | Tool-calling agent + secondary formatting agent | OpenRouter | Tavily Search |
| [Resume Tailoring Agent](workflows/resume-tailoring-agent/README.md) | Document extraction → single-shot rewriting agent | OpenRouter | — |

More workflows will be added here as they're built — see `_template/` to
contribute one.

---

## Prerequisites (all workflows)

- **n8n** — self-hosted or Cloud ([install guide](https://docs.n8n.io/hosting/installation/))
- **OpenRouter API key** — [openrouter.ai](https://openrouter.ai/) (free tier available)
- Workflow-specific API keys are listed in each workflow's own README

## Setup (all workflows)

1. Install n8n.
2. Import the specific `workflow.json` you want from its folder under `workflows/`
   (**Workflows → Import from File** in the n8n editor).
3. Create the credentials listed in that workflow's README under
   **Credentials → New**. Never paste API keys directly into node parameters —
   see [`docs/security.md`](docs/security.md) for why and how.
4. Re-attach credentials on each node after import (n8n does not auto-link
   credentials from an imported file).
5. Toggle the workflow **Active** and open its chat widget URL.

---

## Repo Structure

```
n8n-ai-agent-blueprints/
├── README.md                  # This file — workflow index
├── CONTRIBUTING.md            # How to add a new workflow
├── docs/
│   └── security.md            # Credential handling policy
├── workflows/
│   └── <workflow-name>/
│       ├── workflow.json      # Sanitized, credential-based n8n export
│       ├── README.md          # Architecture, setup, prompts
│       └── sample-data/       # Optional test fixtures
└── _template/                 # Starting point for new workflows
    ├── workflow.json.example
    └── README.template.md
```

## License

Provided as-is for educational and portfolio/demonstration purposes.
