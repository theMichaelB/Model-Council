# Model Council — Round 1: Independent research

You are the **claude** seat on a three-model "Model Council" (seats: `claude`, `codex`, `grok`).
The council's job is to research and plan an **AI coding harness** for one user.

## The user
- A casual coder with ~1 year of AI-assisted coding experience.
- Already understands AI coding limitations and that they are fixed with guardrails, reviews, tests, etc.
- Does **not** want to hand-build that harness. The end goal is a **complete coding harness, built
  entirely by AI**, that reliably produces functional, high-quality code.
- Has: a Claude subscription (Claude Code), a ChatGPT/OpenAI subscription (Codex), a Grok
  subscription, and boat.dev cloud sandboxes (VMs that can run Claude Code and Codex headlessly,
  with snapshots, parallel conversations, and an API/CLI).

## Your task (this round)
Independently research and write a proposal for this harness. Do real research: use web search /
fetch if you have it, and cite sources (URLs) for factual claims. Prefer recent (2025–2026) material.
Cover at least:

1. **State of the art** — how good AI coding harnesses/agent workflows work today, e.g.: agent loops;
   spec-first / plan-first workflows; test-driven guardrails; linters, formatters, type checks;
   CI gates; multi-model / adversarial review; sandboxing and permissions; memory and context
   management (repo instructions files, docs, retrieval); task decomposition and parallel agents;
   evaluating output quality (evals, benchmarks, mutation testing, etc.). Note what actually works
   vs. hype.
2. **Recommended architecture** — the components of the harness, how they fit together, and the
   flow of one feature from idea to merged code. Be concrete about which model/tool plays which role
   given the user's subscriptions and boat.dev.
3. **Components list** — each component, its purpose, and existing tools it could reuse vs. build.
4. **Phased build plan** — where AI builds everything. Small phases, each with a clear deliverable
   and an objective "done" check. Say how the harness bootstraps itself (early phases used to build
   later ones).
5. **Quality bar** — how we will know the harness produces functional, high-quality code
   (metrics, evals, gates).
6. **Risks and failure modes** — and mitigations (cost, runaway agents, reward hacking of tests,
   security, lock-in, subscription rate limits, complexity the user won't maintain).
7. **Open questions for the user** — decisions only the user can make.

Keep it practical for a casual coder: favour boring, reusable tools over clever bespoke ones.
Aim for a thorough but readable document (roughly 1,500–4,000 words), with a short executive
summary at the top.

## Rules
- Work in the git repository at `/home/user/Model-Council` (https://github.com/theMichaelB/Model-Council).
- Write your proposal to `rounds/01-research/claude.md`. You may add supporting notes under
  `rounds/01-research/claude-notes/` if useful. Do not modify anything else.
- **Independence: do NOT read, open, or search other models' files in `rounds/` this round**, and do
  not look at other seats' prompts or transcripts. Your proposal must be your own.
- Commit early and regularly (e.g. after an outline, after research notes, after the full draft),
  with messages prefixed `[claude]`, and `git pull --rebase` then `git push` after each commit (other seats push to the same branch; a rebase that brings in their files is fine, just don't open them). If push fails, keep
  committing locally and say so at the end — the orchestrator will push on your behalf.
- Never write tokens, keys, or credentials into any file or commit.
- When finished, reply with a short summary of your proposal and the list of your commits.
