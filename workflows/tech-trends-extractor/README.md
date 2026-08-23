# Tech Trends Extractor

A conversational agent that researches a tech topic via live web search and
returns a curated, formatted page summarizing what it found.

**Pattern demonstrated:** tool-calling agent (agent decides when/how to call
an external search API) feeding a second, single-purpose formatting agent —
i.e. a lightweight two-agent pipeline rather than one agent doing everything.

> **Update:** this workflow originally ran on OpenRouter free-tier models.
> It's since been migrated to **Google Gemini** (`models/gemini-3.6-flash`)
> after hitting OpenRouter's zero-credit rate limit during testing. See
> [Migration notes](#migration-notes-openrouter--google-gemini) below.

## Architecture

```
[Chat Trigger]
      │
      ▼
[Agent - Tech Trends Extractor]  ──uses──▶  [Google Gemini Chat Model]
      │         │
      │         └──uses tool──▶ [HTTP Request Tool → Tavily Search API]
      │
      └──uses memory──▶ [Simple Memory: buffer window, last 10 messages]
      │
      ▼
[Agent - Webpage curator]  ──uses──▶  [Google Gemini Chat Model1]
      │
      ▼
[HTML node]  → returned to chat
```

## Nodes

| Node | Type | Key config |
|---|---|---|
| `When chat message received` | `chatTrigger` | Default options — provides the built-in n8n chat widget |
| `Agent - Tech Trends Extractor` | `agent` | System message defines a friendly, self-explaining demo persona; instructed to prefer calling tools over reasoning alone |
| `Google Gemini Chat Model` | `lmChatGoogleGemini` | Model: `models/gemini-3.6-flash` |
| `HTTP Request` (tool) | `httpRequestTool` | POST `https://api.tavily.com/search`; body `search_depth: advanced`; query populated by the agent via `$fromAI(...)`; auth via **Header Auth credential** — see [security.md](../../docs/security.md) |
| `Simple Memory` | `memoryBufferWindow` | `contextWindowLength: 10` — short-term, per-session only |
| `Agent - Webpage curator` | `agent` | Prompt: `Generate a html code for the following contents {{ $json.output }}` |
| `Google Gemini Chat Model1` | `lmChatGoogleGemini` | Model: `models/gemini-3.6-flash` |
| `HTML` | `n8n-nodes-base.html` | `html: {{ $json.output }}` — wraps agent output; see limitation below |

## System Prompt (Tech Trends Extractor agent)

```
<role>
You are the n8n Demo AI Agent, a friendly and helpful assistant designed to showcase the power of AI agents within the n8n automation platform. Your personality is encouraging, slightly educational, and enthusiastic about automation. Your primary function is to demonstrate your capabilities by using your available tools to answer user questions and fulfill their requests. You are conversational.
</role>

<instructions>
<goal>
Your primary goal is to act as a live demonstration of an AI Agent built with n8n. You will interact with users, answer their questions by intelligently using your available tools, and explain the concepts behind AI agents to help them understand their potential.
</goal>

<context>
### How I Work
I am an AI model operating within a simple n8n workflow. This workflow gives me two key things:
1.  **A set of tools:** These are functions I can call to get information or perform actions.
2.  **Simple Memory:** I can remember the immediate past of our current conversation to understand context.

### My Purpose
My main purpose is to be a showcase. I demonstrate how you can give a chat interface to various functions (my tools) without needing complex UIs. This is a great way to make powerful automations accessible to anyone through simple conversation.

### My Tools Instructions
You must choose one of your available tools if the user's request matches its capability. You cannot perform these actions yourself; you must call the tool.

### About AI Agents in n8n
- **Reliability:** While I can use one tool at a time effectively, more advanced agents can perform multi-step tasks. However, for complex, mission-critical processes, it's often more reliable to build structured, step-by-step workflows in n8n rather than relying solely on an agent's reasoning. Agents are fantastic for user-facing interactions, but structured workflows are king for backend reliability.
- **Best Practices:** A good practice is to keep an agent's toolset focused, typically under 10-15 tools, to ensure reliability and prevent confusion.

### Current Date & Time
{{ $now }}
</context>

<output_format>
- Respond in a friendly, conversational, and helpful tone.
- When a user's request requires a tool, first select the appropriate tool. Then, present the result of the tool's execution to the user in a clear and understandable way.
- Be proactive. If the user is unsure what to do, suggest some examples of what they can ask you based on your available tools (e.g., Talk about your tools and what you know about yourself).
</output_format>
</instructions>
```

## Credentials required

| Credential | Type | Used by | Get it from |
|---|---|---|---|
| Google Gemini (PaLM) API | Google Gemini credential | Both Chat Model nodes | [aistudio.google.com/apikey](https://aistudio.google.com/apikey) — free tier, no card required |
| Tavily API | Header Auth (`Authorization: Bearer <key>`) | HTTP Request tool node | [tavily.com](https://tavily.com/) |

See [`docs/security.md`](../../docs/security.md) for how to wire the Header
Auth credential — do not paste the key into the node directly. **Important:**
the Header Auth credential's **Name** field must be the literal HTTP header
name (`Authorization`), not a descriptive label — using a descriptive label
there silently breaks the request (see Migration notes below).

## Usage

1. Open the chat widget URL for this workflow.
2. Ask about any tech topic, e.g. *"What are the latest trends in AI agents?"*
3. The agent searches the web via Tavily, then a second agent formats the
   findings.

## Migration notes: OpenRouter → Google Gemini

This workflow originally used OpenRouter (`nvidia/nemotron-3.5-lightning:free`
and `openrouter/free`) as the model provider. Two issues surfaced during
testing that led to switching to Google Gemini:

1. **Invalid model string:** `"openrouter/free"` was not a real, resolvable
   OpenRouter model ID. Requests silently routed to NVIDIA's content-safety
   guardrail model instead of an actual chat model, producing outputs like
   `User Safety: safe` rather than real content. Always verify the exact
   model slug from the provider's own model page before hardcoding it.
2. **Free-tier rate limits:** OpenRouter accounts with **zero credit balance**
   are capped at a very low request rate for `:free` models (roughly
   20 req/day observed), regardless of which specific free model is used.
   Confirmed via OpenRouter's Activity dashboard showing rate-limit errors
   after only 3 total requests.

**Fix applied:** replaced both `OpenRouter Chat Model` nodes with
**Google Gemini Chat Model** nodes (`models/gemini-3.6-flash`), using a
free Google AI Studio API key (1,500 requests/day free tier, no credit card
required). No functional changes to prompts, tools, or memory — only the
LLM provider changed.

If you re-introduce OpenRouter, verify the exact model slug on
[openrouter.ai/models](https://openrouter.ai/models) before use, and
consider adding a small credit balance ($5+) to unlock OpenRouter's higher
free-tier rate limit if you plan to use `:free` models at any real volume.

## Known limitations / production TODOs

- **No error handling.** A Tavily timeout or Gemini rate limit fails the
  turn silently. Add an Error Trigger workflow or IF-node fallback.
- **`HTML` node is a label, not a renderer.** It wraps the curator agent's
  raw output string into an `html` field but doesn't actually serve it as
  rendered HTML. To truly render a page, replace it with a **Respond to
  Webhook** node set to `Content-Type: text/html`, or serve via a static
  route.
- **Two LLM calls per request** (research agent + curator agent) roughly
  doubles token cost/latency vs. a single agent instructed to output HTML
  directly. Kept separate here for clarity of the multi-agent pattern —
  collapse to one agent if optimizing for cost.
- **Memory is per-execution/session only** (buffer window, not persisted).
  For cross-session memory, swap in a Postgres/Redis-backed memory node
  keyed by a session ID from the chat trigger.
