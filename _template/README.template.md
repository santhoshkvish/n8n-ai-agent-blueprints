# <Workflow Name>

<One to two sentences: what does this agent do, end to end?>

**Pattern demonstrated:** <e.g. tool-calling agent / multi-agent pipeline /
RAG retrieval / human-in-the-loop approval / stateful memory>

## Architecture

```
[Trigger]
    │
    ▼
[Step]
    │
    ▼
[Output]
```
<Replace with the actual node chain, including any tools/memory/sub-models
branching off an agent node.>

## Nodes

| Node | Type | Key config |
|---|---|---|
| `<name>` | `<n8n type>` | <parameters/expressions that matter> |

## System Prompt(s)

```
<Full system message(s) used by any agent node, verbatim>
```

## Credentials required

| Credential | Type | Used by | Get it from |
|---|---|---|---|
| | | | |

Remember: **no key ever goes directly into a node parameter.** See
[`docs/security.md`](../../docs/security.md).

## Sample data

<If applicable — what's in sample-data/ and how to use it>

## Usage

1.
2.
3.

## Known limitations / production TODOs

- <error handling gaps>
- <rate limit / model stability caveats>
- <anything simplified for demo purposes that you'd change for real traffic>
