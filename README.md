# Model Council

A council of three AI models that researches and plans an **AI coding harness**: a system that
lets a casual coder get functional, high-quality code out of AI agents by wrapping them in
guardrails (specs, tests, linters, CI gates, multi-model review, sandboxing) — with the harness
itself built entirely by AI.

## Members

| Seat | Model / harness | How it runs |
|------|-----------------|-------------|
| `claude` | Anthropic Claude via **Claude Code** | boat.dev sandbox, user's Claude subscription |
| `codex`  | OpenAI GPT via **Codex CLI** | boat.dev sandbox, user's ChatGPT subscription |
| `grok`   | xAI Grok (the orchestrating bot, "Model Council") | runs on the orchestrator's box |

Grok also acts as **orchestrator**: it sends prompts, logs them, exports transcripts and commits
on behalf of a model when that model cannot push itself.

## Process

1. **Independent research** — each model researches and writes its own proposal without reading
   the others' (`rounds/01-research/<model>.md`).
2. **Cross-critique** — each model reads the other two proposals and critiques them
   (`rounds/02-critique/`).
3. **Synthesis** — the strongest ideas are merged into one plan, with disagreements listed
   (`rounds/03-synthesis/`).
4. **Final review** — Claude and GPT each sign off or flag problems (`rounds/04-review/`).

## Layout

```
prompts/PROMPT_LOG.md            append-only log of every prompt sent to every model
prompts/<round>/<model>.md       the exact prompt file sent
rounds/<round>/<model>.md        each model's output for that round
transcripts/<round>/<model>/     raw chat history exported verbatim (session JSONL, boat events)
```

## Rules

- Every model commits its own work with a message prefixed by its seat, e.g. `[claude] ...`.
  If a sandbox cannot push, the orchestrator commits on its behalf with the same prefix.
- Prompts are logged **before** they are sent.
- Transcripts are raw and unedited, except that any secret/token is redacted as `[REDACTED]`.
- Times are Europe/London.
