# Grok seat — Round 1 raw research log

The Grok seat is the orchestrating bot itself. It has no exportable session file like Claude Code
JSONL or a Codex rollout. This is a verbatim list of the tool calls it made while writing
`rounds/01-research/grok.md`, in order, between 04:39 and 04:44 Europe/London (GMT) on 17 Nov
2026. Per the independence rule, it did not open other seats' proposals before committing its own.

## Prompt
See `prompts/01-research/grok.md` (logged in `prompts/PROMPT_LOG.md` before work began).

## Web searches (query → top sources used)
1. `Anthropic building effective agents harness long-running agents 2026` → anthropic.com/research/harness-design-long-running-apps, /effective-harnesses-for-long-running-agents, github.com/anthropics/cwc-long-running-agents, /research/managed-agents
2. `spec-driven development AI coding agents GitHub Spec Kit Kiro 2026` → github.com/github/spec-kit, webdevdrops.com/spec-driven-development, umar.codes/spec-driven-development-tools, paperclipped.de Kiro review
3. `AI coding agents reward hacking tests deleting tests study 2025 2026` → arxiv 2511.21654 (EvilGenie), arxiv 2611.10404 (Coding or Cheating?), gemini-cli PR #9253, Georgia Tech thesis
4. `METR study AI developers productivity experienced open source 2025 results` → metr.org 2025-07-10 study, metr.org 2026-02-24 uplift update
5. `AGENTS.md standard repository instructions coding agents` → agents.md, github.blog agents.md lessons, learn.chatgpt.com AGENTS.md docs
6. `Claude Code hooks headless mode subagents documentation` → code.claude.com/docs/en/hooks, /sub-agents, /headless
7. `Codex CLI exec headless code review /review GitHub Action 2026` → github.com/openai/codex-action, developers.openai.com Codex SDK code review cookbook
8. `Simon Willison lethal trifecta prompt injection coding agents` → simonwillison.net 2025/jun/16, 2025/Sep/26, 2025/Sep/30
9. `Meta mutation-guided LLM test generation ACH mutation testing` → arxiv 2501.12862, engineering.fb.com ACH posts
10. `cross-model code review LLM different model reviewer catches more bugs study` → stephens.page "Fresh eyes beat a second family", arxiv 2610.01471, arxiv 2603.12123, arxiv 2607.21656
11. `GitClear AI code quality report duplication churn 2025` → gitclear.com 2025 research
12. `xAI Grok Build coding agent CLI 2026` → docs.x.ai/build/overview, docs.x.ai/build/cli/headless-scripting
13. `DORA 2025 State of AI-assisted Software Development report key findings` → dora.dev/research/2025/dora-report
14. `Anthropic effective context engineering for AI agents` → anthropic.com/research/effective-context-engineering-for-ai-agents
15. `property-based testing LLM agents Hypothesis catches bugs 2025 paper` → arxiv 2510.09907, anthropic.com/research/property-based-testing

## Pages fetched in full
- https://docs.x.ai/build/overview (Grok Build install, headless mode, XAI_API_KEY auth)
- https://arxiv.org/html/2607.21656 (cross-model review direction results)

## Output
- `rounds/01-research/grok.md`, commit `0bcba5b` "[grok] Round 1: independent research proposal"
