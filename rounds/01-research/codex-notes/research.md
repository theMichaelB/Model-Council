# Independent research ledger

Researched 2026-11-17. No other council seat's round files, prompts, or transcripts were opened. Sources are primary documentation/research; vendor case studies are not controlled evidence of general reliability.

## Sources and implications

- Anthropic, 2025-11-26: https://www.anthropic.com/research/effective-harnesses-for-long-running-agents — initialization, incremental work, progress artifacts, and real browser testing; useful pattern, not a guarantee.
- Anthropic, 2026-03-24: https://www.anthropic.com/research/harness-design-long-running-apps — planner/generator/evaluator pattern; QA required calibration; newer models reduced the need for orchestration.
- OpenAI, 2026-02-11: https://openai.com/index/harness-engineering/ — short instruction index, versioned repository knowledge, mechanical architectural enforcement. Internal experience, not a randomized comparison.
- Anthropic, 2025-06-13: https://www.anthropic.com/research/multi-agent-research-system — parallelism helps separable research; communication and token costs matter; do not generalize its research results to coding teams.
- Anthropic, 2026-01-09: https://www.anthropic.com/research/demystifying-evals-for-ai-agents — evaluate outcomes, multiple trials, code/model/human graders.
- METR, 2025 study: https://arxiv.org/abs/2507.09089 — 16 experienced developers, 246 tasks, early-2025 tools; 19% slowdown. Narrow population and dated tools.
- METR, 2026-02-24: https://metr.org/blog/2026-02-24-uplift-update/ — selection problems undermine simple newer productivity estimates; avoid treating the old slowdown as timeless.
- OpenAI, 2026-02-23: https://openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/ — contamination and test defects limit benchmark interpretation.
- Codex automation: https://learn.chatgpt.com/docs/non-interactive-mode — exec, structured/event outputs, saved authentication; protect auth like passwords; account auth CI has additional constraints.
- Claude automation: https://code.claude.com/docs/en/headless — native CLI programmatic mode, tool restrictions, structured output.
- Claude credentials: https://code.claude.com/docs/en/legal-and-compliance — native subscription use versus API/Agent SDK credentials; third-party OAuth routing is not permitted.
- xAI billing: https://docs.x.ai/grok/faq and https://docs.x.ai/x-subscription/overview — current first-party usage offerings have changed; account/API entitlement must be verified, not inferred from an unspecified subscription.
- Boat: https://boat.dev/ — snapshots, forks, CLI and API/SDK links; disk persistence is not process persistence or permanent backup.
- GitHub: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches — branch gates, permissions, plan availability; skipped checks can satisfy required status checks, so trusted aggregate verification matters.
- Mutation metrics: https://stryker-mutator.io/docs/mutation-testing-elements/mutant-states-and-metrics/ — killed/survived/no-coverage distinctions; a green coverage percentage does not show assertions detect faults.

## Local verification (no credentials inspected)

- Installed Codex: `codex-cli 0.169.0`; `codex exec --help` confirms noninteractive execution, sandbox policies, managed worktree option.
- `boat --help` confirms prompt, fork, snapshot/stop/resume, API execution, limits, and usage commands. `boat fork --help` exposes TTL and no-env controls. No VM was created.
- Read installed sandbox skill and official-docs skill; refreshed official Codex manual to temporary cache and inspected relevant CLI/authentication excerpts. Do not copy credentials from a personal HOME into worker templates.

## Design decisions (proposals, not source claims)

Use TypeScript + SQLite + a small deterministic state machine; GitHub as user interface; first implementation sequential; no vector database, agent framework, or custom dashboard. Treat multi-model review as optional added evidence, not a majority vote. Fail closed on missing checks; keep policy, evaluator, and merge authority outside worker write access. Prove the bootstrap with fault injection before permitting automation to build later phases.
