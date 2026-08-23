# Tech Trends Extractor

A conversational agent that researches a tech topic via live web search and
returns a curated, formatted page summarizing what it found.

**Pattern demonstrated:** tool-calling agent (agent decides when/how to call
an external search API) feeding a second, single-purpose formatting agent —
i.e. a lightweight two-agent pipeline rather than one agent doing everything.

## Architecture

```
[Chat Trigger]
      │
      ▼
[Agent: Tech Trends Extractor]  ──uses──▶  [OpenRouter Chat Model: nvidia/nemotron-3.5-lightning:free]
      │         │
      │         └──uses tool──▶ [HTTP Request Tool → Tavily Search API]
      │
      └──uses memory──▶ [Simple Memory: buffer window, last 10 messages]
      │
      ▼
[Agent: Webpage Curator]  ──uses──▶  [OpenRouter Chat Model1: openrouter/free]
      │
      ▼
[HTML node]  → returned to chat
```

## Nodes

| Node | Type | Key config |
|---|---|---|
| `When chat message received` | `chatTrigger` | Default options — provides the built-in n8n chat widget |
| `Agent - Tech Trends Extractor` | `agent` | System message defines a friendly, self-explaining demo persona; instructed to prefer calling tools over reasoning alone |
| `OpenRouter Chat Model` | `lmChatOpenRouter` | Model: `nvidia/nemotron-3.5-lightning:free` |
| `HTTP Request` (tool) | `httpRequestTool` | POST `https://api.tavily.com/search`; body `search_depth: advanced`; query populated by the agent via `$fromAI(...)`; **auth via Header Auth credential, not a hardcoded key — see [security.md](../../docs/security.md)** |
| `Simple Memory` | `memoryBufferWindow` | `contextWindowLength: 10` — short-term, per-session only |
| `Agent - Webpage Curator` | `agent` | Prompt: `Generate a html code for the following contents {{ $json.output }}` |
| `OpenRouter Chat Model1` | `lmChatOpenRouter` | Model: `openrouter/free` |
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
| OpenRouter API | OpenRouter API | Both Chat Model nodes | [openrouter.ai](https://openrouter.ai/) |
| Tavily API | Header Auth (`Authorization: Bearer <key>`) | HTTP Request tool node | [tavily.com](https://tavily.com/) |

See [`docs/security.md`](../../docs/security.md) for how to wire the Header
Auth credential — do not paste the key into the node directly.

## Usage

1. Open the chat widget URL for this workflow.
2. Ask about any tech topic, e.g. *"What are the latest trends in AI agents?"*
3. The agent searches the web via Tavily, then a second agent formats the
   findings.

## Known limitations / production TODOs

- **No error handling.** A Tavily timeout or OpenRouter rate limit fails the
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
- **`openrouter/free`** routing and rate limits change over time; fine for
  a demo, not for production traffic.
- **Memory is per-execution/session only** (buffer window, not persisted).
  For cross-session memory, swap in a Postgres/Redis-backed memory node
  keyed by a session ID from the chat trigger.
