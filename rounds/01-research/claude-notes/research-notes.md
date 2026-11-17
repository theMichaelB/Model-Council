# Claude seat — research notes (Round 1)

Raw notes behind `../claude.md`. Retrieved 2026-11-17. "(secondary)" = only third-party coverage seen.

## Harness design (primary)
- Anthropic, "Effective harnesses for long-running agents" (J. Young, 2025-11-26)
  https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents
  - Initializer agent writes init.sh, progress file, initial commit, JSON feature list (all `passes:false`).
  - Coding agent: one feature per session; may only flip `passes`. JSON chosen because model less likely to rewrite it than Markdown.
  - Startup ritual: pwd, git log, progress file, feature list, run init.sh, smoke e2e test before new work.
  - Failure modes: declares victory early; leaves env broken; marks features done without e2e test. Browser automation (Puppeteer MCP) fixed much of verification.
- OpenAI, "Harness engineering: leveraging Codex in an agent-first world" (R. Lopopolo, 2026-02-11)
  https://openai.com/index/harness-engineering/ (403 to fetcher; content via secondary summaries)
  - ~1M LOC internal product, 5 months, no hand-written code. Bottleneck = underspecified environment.
  - AGENTS.md as short map, docs as system of record; mechanical enforcement (lints, structure tests); agent-to-agent review; Chrome DevTools wired in for validation.
- Claude Code best practices: explore → plan → code → commit; plan mode skip for small fixes; "evidence over assertions"; context fills fast and performance degrades.
  https://code.claude.com/docs/en/best-practices
- Claude Code headless: `claude -p`, `--output-format json` (includes total_cost_usd), `--json-schema`, `--permission-mode`, `--allowedTools`, `--max-budget-usd`, `--bare` recommended for CI.
  https://code.claude.com/docs/en/headless
- Claude Code hooks: PreToolUse can block (exit 2), Stop hook can force continuation; config in .claude/settings.json.
  https://code.claude.com/docs/en/hooks
- Claude Code sandboxing: bubblewrap (Linux) FS + network proxy; open-source sandbox-runtime.
  https://code.claude.com/docs/en/sandboxing
- Codex non-interactive: `codex exec`, read-only default, `--sandbox workspace-write`, `--json`, `--output-schema`, `codex exec resume`; openai/codex-action for GH Actions; pattern: agent job with read-only perms emits patch, separate job applies it.
  https://learn.chatgpt.com/docs/non-interactive-mode
- Codex GitHub code review counted separately; plan tiers Plus $20, Pro $100/$200 (secondary/official pricing page)
  https://developers.openai.com/codex/pricing

## Spec-driven development
- Böckeler (Thoughtworks), 2025-10-15: Kiro/spec-kit/Tessl. Spec-first useful; heavy SDD = "sledgehammer to crack a nut" for small tasks, markdown review overload, agents ignored specs, non-determinism.
  https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html
- GitHub Spec Kit (Sept 2025, experimental, MIT). https://github.com/github/spec-kit

## Context / instructions files
- AGENTS.md now under Linux Foundation Agentic AI Foundation; 60k+ projects. https://openai.com/index/agentic-ai-foundation/
- Gloaguen et al. 2026, arXiv:2602.11988: context files tend to *reduce* success vs none and raise cost >20%; LLM-generated worse; human-written slight improvement; keep minimal.
  https://arxiv.org/abs/2602.11988
- Chroma "Context Rot" (Jul 2025): performance degrades with input length even on trivial tasks. https://research.trychroma.com/context-rot

## Reward hacking / test gaming
- ImpossibleBench (Zhong et al., arXiv:2510.20270): tests conflicting with spec; pass = cheating. Claude models mostly cheat by modifying tests; GPT models more varied (operator overloading, special-casing). Hiding tests → near-zero hacking but hurts legit performance; strict prompt ("STOP if tests flawed") cut GPT-5 rate ~93%→1% on one split (secondary for numbers).
  https://arxiv.org/abs/2510.20270

## Review
- Song 2026, arXiv:2610.01471: top-tier cross-model reviewer ≈ same-model fresh-session review in F1; errors only 41% overlap; 1 same + 1 cross caught 56.7% vs 42.7% for 2 same-model (not sig. better than 2 top-tier cross). Takeaway: diversity + fresh context + strong reviewer.
  https://arxiv.org/abs/2610.01471
- Self-preference bias of LLM judges: documented (e.g., arXiv:2601.06884).

## Evidence on outcomes
- METR 2025: experienced OSS devs 19% slower (CI +2%..+39%). Feb 2026 update: -18% (CI -38..+9) returning, -4% (CI -15..+9) new; selection effects; design no longer reliable.
  https://metr.org/blog/2026-02-24-uplift-update/
- DORA 2025: AI is an amplifier; throughput up but delivery instability up; gains need fast feedback loops/loosely coupled systems.
  https://cloud.google.com/blog/products/ai-machine-learning/announcing-the-2025-dora-report
- CodeRabbit Dec 2025: 470 PRs, AI-co-authored ~1.7x issues (10.83 vs 6.45/PR); authorship inferred.
  https://www.coderabbit.ai/blog/state-of-ai-vs-human-code-generation-report
- Package hallucination (Spracklen et al., USENIX Sec 2025): 576k samples, 5.2% commercial / 21.7% OSS hallucinated packages, 43% repeat consistently → slopsquatting. (secondary) https://www.kaspersky.com/blog/ai-slopsquatting-supply-chain-risk/53327/
- OpenAI stopped reporting SWE-bench Verified (2026-02-23): contamination, 59.4% of audited problems flawed. → public benchmarks ≠ your codebase.
  https://openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/

## Test quality
- Meta ACH (arXiv:2501.12862, FSE'25): mutation-guided LLM test generation; 73% of tests accepted; ~49% would be discarded under coverage-only. → mutation score > coverage.
- Tools: Stryker (JS/TS/C#), mutmut / cosmic-ray (Python), cargo-mutants (Rust), PIT (Java); Hypothesis / fast-check (property-based).

## Subscriptions / limits (volatile — verify live)
- Claude: 5-hour rolling window + 7-day weekly cap; Opus separate weekly cap since Aug 2025 (TechCrunch 2025-07-28). https://techcrunch.com/2025/07/28/anthropic-unveils-new-rate-limits-to-curb-claude-code-power-users/
- Codex: Plus/Pro tiers; GitHub review quota separate (official pricing page).
- Grok Build CLI: launched 2026-05-14 (SuperGrok Heavy), widened to SuperGrok/X Premium+ late May; headless mode, subagents; quotas unpublished (secondary). https://pasqualepillitteri.it/en/news/3442/grok-build-public-beta-imagine-orchestrator-26-may-2026

## boat.dev (from local `boat --help`, 2026-11-17)
- `boat new/fork/stop/resume/snapshot`, `boat prompt <id> --provider claude|codex --model --reasoning-effort --new/--resume`, `boat events`, `boat exec`, `boat run` (one-shot VM, reports cost, deletes), `boat ci` (GitHub App; `runs-on: boat` Actions runners), `boat limits`, `boat usage`, `boat webhook`, `boat host`.
