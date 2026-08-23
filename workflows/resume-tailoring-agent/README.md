# Resume Tailoring Agent

Accepts a job description (chat text) plus a resume (PDF upload), and
returns a fully rewritten, ATS-optimized resume tailored to that job.

**Pattern demonstrated:** file upload → text extraction → single-shot
(stateless, tool-free) rewriting agent. Simpler than the Tech Trends agent
by design — this is a document-transform task, not an open-ended research
task, so it doesn't need tools or memory.

## Architecture

```
[Chat Trigger, allowFileUploads: true]
      │
      ▼
[Extract from File: pdf]
      │
      ▼
[Agent: Resume Tailoring Agent]  ──uses──▶  [OpenRouter Chat Model: openrouter/free]
      │
      ▼
[HTML node]  → returned to chat
```

## Nodes

| Node | Type | Key config |
|---|---|---|
| `When chat message received` | `chatTrigger` | `public: true`, `allowFileUploads: true` — enables the upload control in the chat widget; custom initial message prompting for the job description |
| `Extract from File` | `extractFromFile` | `operation: pdf`, `binaryPropertyName: {{ $json.files[0].binaryKey }}` — pulls the first uploaded file's binary content |
| `Resume Generator Agent` | `agent` | User message combines job description (pulled from the **trigger node by name**) and extracted resume text; full ATS-tailoring system prompt (below) |
| `OpenRouter Chat Model` | `lmChatOpenRouter` | Model: `openrouter/free` |
| `HTML` | `n8n-nodes-base.html` | `html: {{ $json.output }}` — see limitation below (output is Markdown, not HTML) |

### Expression pattern worth noting

The agent's user message pulls two different upstream values two different ways:

```
<job description>
{{ $('When chat message received').item.json.chatInput }}
</job description>

<resume>
{{ $json.text }}
</resume>
```

`chatInput` is referenced **explicitly by trigger node name** because it's
needed further downstream than the immediately preceding node. `$json.text`
uses plain chaining because it comes from the node directly before this one
(`Extract from File`). When a value is needed beyond the immediate
predecessor, reference the source node by name — don't rely on `$json`
chaining across multiple hops.

## System Prompt (Resume Tailoring Agent)

```
<role>
You are the Resume Tailoring Agent, a professional and meticulous career optimization assistant. Your personality is precise, strategic, and deeply knowledgeable about hiring practices, ATS (Applicant Tracking System) optimization, and resume best practices. Your primary function is to receive a candidate's existing resume content and a target job description, then produce a newly tailored resume that maximizes the candidate's alignment with the role.
</role>

<instructions>
<goal>
Your primary goal is to generate a fully rewritten, job-targeted resume from two inputs: the candidate's current resume content and the target job description. The tailored resume must:
1. Preserve and reframe as much of the candidate's original experience, skills, projects, and achievements as possible — never discard information unless it is entirely irrelevant.
2. Align language, keywords, and emphasis with the job description to maximize ATS compatibility and recruiter relevance.
3. Add complementary skills and experience only when they closely align with the candidate's existing background and are clearly demanded by the job description. Any additions must be plausible extensions of the candidate's demonstrated expertise — never fabricate unrelated experience.
</goal>

<context>
### Tailoring Strategy
1. Analyze the job description — required/preferred skills, responsibilities, domain keywords, ATS-critical exact phrases.
2. Map the candidate's resume to the job — identify matches, gaps, and transferable experience.
3. Rewrite and tailor — incorporate keywords naturally, reorder for relevance, strengthen quantifiable achievements, adjust the summary, expand the skills section to mirror JD terminology.
4. Fill strategic gaps — add plausible entries only where there's adjacent expertise; never fabricate unrelated experience.
5. Final polish — consistent formatting/tense/tone, verify ATS keywords present, remove filler ("responsible for" → "led").

### Rules
- Accuracy over embellishment. Stretch is acceptable; fabrication is not.
- Maintain the candidate's voice/tone unless clearly unprofessional.
- Retain unique differentiators (projects, publications, niche skills) even if not explicitly in the JD.
- ATS optimization is mandatory: standard section headings, no tables/columns/graphics, natural keyword embedding.
</context>

<output_format>
- Complete tailored resume in clean Markdown: Professional Summary, Skills, Experience, Education, plus relevant extras.
- Each experience entry: Job Title, Company, Dates, bullet points.
- A "Tailoring Notes" section after a horizontal rule: keywords injected, what was rewritten/reordered, any additions with justification, remaining gaps for the candidate to address in interview/cover letter.
</output_format>
</instructions>
```

## Credentials required

| Credential | Type | Used by | Get it from |
|---|---|---|---|
| OpenRouter API | OpenRouter API | Chat Model node | [openrouter.ai](https://openrouter.ai/) |

## Sample data

Two test resumes are provided in `sample-data/`:
- `Alex Turing_resume.pdf`
- `Jordan Patel_resume.pdf`

A sample job description to test against is in the root project's original
`Instructions.md` history (a senior AI/ML engineering role) — any real JD
works equally well.

## Usage

1. Open the chat widget URL for this workflow.
2. Paste a job description as your message.
3. Upload a resume PDF via the chat's file-upload control.
4. Receive the tailored resume + Tailoring Notes.

## Known limitations / production TODOs

- **Output is Markdown, not HTML**, despite the trailing `HTML` node — that
  node currently just labels the field, it doesn't convert Markdown to
  rendered HTML. Either drop it (n8n's chat widget renders Markdown
  natively) or add a real Markdown→HTML conversion step if you need an
  actual HTML artifact.
- **No error handling** if the PDF extraction fails (e.g. a scanned/image-only
  PDF with no extractable text layer) — the agent will silently receive an
  empty resume. Add a validation/IF branch after `Extract from File`.
- **Single-file assumption** — `files[0]` only ever reads the first upload;
  if a user attaches multiple files, the rest are ignored with no warning.
- **Stateless per-request** — no memory node, so multi-turn refinement
  ("make it more concise") isn't supported as-is; would need a memory node
  added if that UX is wanted.
