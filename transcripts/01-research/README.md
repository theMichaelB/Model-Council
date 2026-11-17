# Round 01 transcripts (raw)

| Seat | File | What it is |
|------|------|------------|
| claude | `claude/claude-code-session-8dff0c32.jsonl` | Native Claude Code session JSONL, copied verbatim from sandbox `bx_gh5vyw57` (`~/.claude/projects/-home-user/`) |
| claude | `claude/boat-events.jsonl` | `boat events bx_gh5vyw57 --json` export (prompt + responses + tool calls) |
| claude | `claude/boat-prompt-stream.jsonl` | Live stdout of `boat prompt` for the turn |
| codex | `codex/codex-rollout-2026-11-17T04-38-50-01a1e7cc.jsonl` | Native Codex rollout JSONL from sandbox `bx_kwasgz9t` (`~/.codex/sessions/2026/11/17/`) |
| codex | `codex/boat-events.jsonl` | `boat events bx_kwasgz9t --json` export |
| codex | `codex/boat-prompt-stream.jsonl` | Live stdout of `boat prompt` for the turn |
| grok | `grok/research-log.md` | Orchestrator's own search/fetch log (no native session file exists) |

Models: Claude Code `opus-5-5` (reasoning effort high); Codex `gpt-6.1-sol` (reasoning effort high).
Conversation ids: claude `dedda394-ea44-43e0-93a1-03c31571c9cf`, codex `530499fd-75f9-4003-a0d7-eb3b19fadc68`.

Secret scan: before committing, the files were scanned for GitHub, OpenAI, Anthropic and xAI
tokens, JWTs, bearer tokens and OAuth env values. Nothing was found and nothing was redacted.

Independence note: when Claude ran `git pull --rebase`, Claude Code's session metadata
(`toolUseResult.bashEditDiff`) recorded the other seats' incoming files. That content never
entered the model's messages. A check confirmed that neither the Claude nor the Codex model
context contains another seat's proposal text.
