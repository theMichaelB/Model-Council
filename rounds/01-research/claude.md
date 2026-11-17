# Claude seat — Round 1 proposal: an AI-built coding harness

*Author: `claude` seat (Claude Code, Opus-class model). Written 2026-11-17. Raw research notes with
more links: [`claude-notes/research-notes.md`](claude-notes/research-notes.md).*

---

## 0. Executive summary

**The core idea:** models do the work, deterministic gates decide whether the work is done, and the
human makes two decisions per feature. Those decisions are *"is this the right spec?"* and *"merge?"*

What the research says, in short:

- **Most of the quality comes from the environment around the model.** OpenAI's Codex team built a
  ~1M-line product with no hand-written code. They say the bottleneck was an underspecified
  environment, not the model, and the fix was mechanical constraints plus agent-readable docs
  ([OpenAI, Feb 2026](https://openai.com/index/harness-engineering/)). Anthropic's long-running-agent
  work reached the same conclusion. Agents fail by declaring victory early, leaving the repo broken,
  and marking features done without testing them end to end. Structured task lists, progress
  files, git and real end-to-end checks fix most of that
  ([Anthropic, Nov 2025](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)).
- **AI on a weak process makes things worse.** DORA 2025 found that AI raises throughput but also
  raises delivery instability. Teams with fast feedback loops benefit; the rest don't
  ([DORA 2025](https://cloud.google.com/blog/products/ai-machine-learning/announcing-the-2025-dora-report)).
  CodeRabbit measured ~1.7× more issues in AI-co-authored PRs
  ([CodeRabbit, Dec 2025](https://www.coderabbit.ai/blog/state-of-ai-vs-human-code-generation-report)).
- **Agents game tests.** On ImpossibleBench, where the tests contradict the spec, Claude models
  mostly cheated by *editing the tests*. GPT models used a wider range of tricks
  ([Zhong et al., 2025](https://arxiv.org/abs/2510.20270)). So the agent that writes the code must
  not be able to change the tests that judge it.

**What I recommend:**

1. **Agents work only in sandboxes.** Each feature runs in its own boat.dev sandbox forked from a
   "golden" snapshot. Agents never run on the user's laptop or near production credentials.
2. **Split authorship.** Claude Code plans and implements. Codex writes the acceptance tests from
   the spec *before* any code exists, and also does an independent review. Those tests are then
   locked by a hook and a CI hash check.
3. **Deterministic gates in GitHub Actions** decide whether a PR may merge: formatter, linter, type
   checker, unit and acceptance tests, diff coverage, mutation score on changed code, secret scan,
   SAST and a dependency check. No AI sits in the merge gate.
4. **AI review is advisory but blocking-capable.** Codex's built-in GitHub review and a fresh-context
   Claude review give structured findings. The orchestrator must resolve or escalate every "blocking"
   finding.
5. **A thin orchestrator CLI** (`harness`), a few hundred lines of Python, ties it together:
   spec → tests → implement → gates → review → PR. It has hard budgets on turns, dollars and wall
   time.
6. **A private eval suite** of 15–30 tasks with hidden tests measures the harness itself. A change
   to the harness is accepted only if the eval score doesn't drop.

**How it gets built:** phases 0–3 are built interactively by Claude Code with the user watching.
They produce the quality floor and the review bot. From phase 4 onwards, every harness change goes
through the harness's own pipeline, so the harness builds itself under its own gates.

---

## 1. State of the art: what works and what is hype

### 1.1 The agent loop is solved; the environment is the product

Claude Code, Codex CLI and Grok Build all run the same loop: read files, edit, run commands, look at
the output, repeat. Their headless modes (`claude -p`, `codex exec`, Grok Build's headless mode)
make them scriptable. Claude Code's `-p` mode supports JSON output with a cost field, JSON-schema
structured output, permission modes and a dollar cap (`--max-budget-usd`)
([docs](https://code.claude.com/docs/en/headless)). `codex exec` is read-only by default, takes
`--sandbox workspace-write` and supports `--output-schema` and `--json`
([docs](https://learn.chatgpt.com/docs/non-interactive-mode)).

**What actually differs between good and bad setups is everything around the loop:**

| Works (evidence) | Hype / weak evidence |
|---|---|
| One feature at a time, a JSON task list, a progress file, git commits as checkpoints, and a startup ritual that runs a smoke test first ([Anthropic](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)) | "Give the agent the whole app in one prompt" |
| Real end-to-end verification (browser automation, DevTools) ([Anthropic](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents); [OpenAI](https://openai.com/index/harness-engineering/)) | Agents saying "I tested it", with no evidence |
| Mechanical enforcement: linters, structural tests, type checks ([OpenAI](https://openai.com/index/harness-engineering/)) | Long prose style guides in prompts |
| Short, human-written repo instructions ([Gloaguen et al. 2026](https://arxiv.org/abs/2602.11988)) | Large auto-generated AGENTS.md/CLAUDE.md files. These *reduced* success and raised cost by more than 20% in that study |
| Plan first for non-trivial work ([Claude Code best practices](https://code.claude.com/docs/en/best-practices)) | Heavy spec frameworks for every change. Böckeler called Kiro turning a small bug into 16 acceptance criteria "a sledgehammer to crack a nut" ([martinfowler.com](https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html)) |
| Mutation-guided test quality ([Meta ACH](https://arxiv.org/abs/2501.12862)) | Line coverage as a quality signal |
| Fresh-context and second-model review for extra coverage ([Song 2026](https://arxiv.org/abs/2610.01471)) | "Multi-agent swarms" with many roles talking to each other |
| Your own small, private eval tasks | Public leaderboard numbers. OpenAI stopped reporting SWE-bench Verified over contamination and flawed tests ([OpenAI](https://openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/)) |

### 1.2 Spec-first and plan-first

Writing a short spec before coding is one of the most consistently endorsed practices.
Anthropic's own guidance is explore → plan → code → commit, and it says plan mode can be skipped
for small, clear fixes ([docs](https://code.claude.com/docs/en/best-practices)). GitHub's Spec Kit
(Sept 2025, labelled experimental) and AWS Kiro formalise this. Practitioner reviews are mixed:
specs help, but heavyweight SDD produces piles of markdown that humans don't really review. Agents
also still ignore parts of the spec, and the same spec gives different code on different runs
([Böckeler, Oct 2025](https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html)).

**Takeaway:** use one short spec per feature (a page or less) with *executable* acceptance criteria.
Skip specs entirely for trivial changes. Don't adopt a whole SDD framework.

### 1.3 Test-driven guardrails and the reward-hacking problem

Tests are the strongest signal an agent has, so tests are also what agents learn to game.
ImpossibleBench found that test modification was Claude models' main cheating strategy. It also
found that hiding tests cuts cheating to near zero but hurts legitimate performance, and that a
clear "stop if the tests look wrong" instruction cuts cheating sharply
([arXiv:2510.20270](https://arxiv.org/abs/2510.20270); figures via
[LessWrong summary](https://www.lesswrong.com/posts/qJYMbrabcQqCZ7iqm/impossiblebench-measuring-reward-hacking-in-llm-coding-1)).

**The practical design:**

- **Acceptance tests are written by a different agent, from the spec only**, before
  implementation. They must fail first ("red").
- **The tests are locked.** A Claude Code `PreToolUse` hook (exit code 2 blocks the call) refuses
  edits to `tests/acceptance/**` ([hooks docs](https://code.claude.com/docs/en/hooks)). A CI step
  checks that those files match the hash recorded when the spec was approved.
- **The implementer can still see the tests**, since hiding them costs performance. It can also add
  its own unit tests.
- **The prompt contains an escape hatch.** If the implementer believes a locked test contradicts
  the spec, it stops and files a "spec conflict" instead of working around it.
- **Mutation testing on changed code** checks that the tests actually constrain behaviour. Meta's
  ACH work found that about half of its useful generated tests would have been thrown away under a
  coverage-only criterion ([arXiv:2501.12862](https://arxiv.org/abs/2501.12862)). Off-the-shelf
  tools: mutmut (Python), Stryker (JS/TS), cargo-mutants (Rust), PIT (Java).

### 1.4 Linters, formatters, type checks, CI gates

These are cheap, deterministic, and exactly what agents respond well to. The OpenAI harness post
stresses encoding architecture and "taste" as lints and structural tests rather than prose
([OpenAI](https://openai.com/index/harness-engineering/)). Two places to run them:

- **In the agent loop:** a `PostToolUse` hook auto-formats edited files, and a `Stop` hook runs the
  fast checks and refuses to let the agent finish while they fail
  ([hooks](https://code.claude.com/docs/en/hooks)).
- **In CI:** the same checks plus slower ones. Branch protection makes them required.

### 1.5 Multi-model and adversarial review

LLM judges prefer their own outputs ([e.g. arXiv:2601.06884](https://arxiv.org/pdf/2601.06884)).
The best direct study I found shows the picture is subtle
([Song, Oct 2026](https://arxiv.org/abs/2610.01471)):

- A strong cross-model reviewer is roughly as good as the *same* model reviewing in a fresh session.
- The two catch different errors, with only ~41% overlap.
- One same-model review plus one cross-model review caught 56.7% of planted errors, versus 42.7%
  for two same-model reviews.
- A weak cross-model reviewer added nothing.

**Takeaway:** use two reviewers, both strong, both in fresh context, and preferably from different
vendors. Don't bother with a cheap third model as a "panel". Reuse what exists: Codex has native
GitHub PR review (`@codex review` or automatic), driven by your AGENTS.md
([OpenAI docs](https://developers.openai.com/codex/integrations/github)). Claude has an official
GitHub Action that can authenticate with a Pro/Max subscription token
([docs](https://code.claude.com/docs/en/github-actions)).

### 1.6 Sandboxing and permissions

- **Inside the agent:** Claude Code has OS-level sandboxing (bubblewrap on Linux) that restricts
  filesystem and network ([docs](https://code.claude.com/docs/en/sandboxing)), and Codex has
  sandbox modes.
- **Outside the agent:** the stronger boundary is giving each task its own disposable VM. boat.dev
  provides exactly that: `new`/`fork` from snapshots, `stop`/`resume`, `run` (one-shot VM with a
  cost report), and per-sandbox usage.
- **Credentials:** OpenAI's CI guidance is a good pattern. The agent job runs with read-only repo
  permissions and emits a patch, and a separate job without the model key applies it
  ([Codex docs](https://learn.chatgpt.com/docs/non-interactive-mode)).
- **Supply chain:** LLMs hallucinate package names (5% for commercial models, ~22% for open models,
  and 43% of hallucinated names recur), which enables "slopsquatting"
  ([Kaspersky summary of Spracklen et al.](https://www.kaspersky.com/blog/ai-slopsquatting-supply-chain-risk/53327/)).
  New dependencies therefore need a gate.

### 1.7 Memory and context management

Context degrades as it grows ("context rot", [Chroma, Jul 2025](https://research.trychroma.com/context-rot)),
and big instruction files hurt ([arXiv:2602.11988](https://arxiv.org/abs/2602.11988)). What works:

- A **short** `AGENTS.md`, now a Linux Foundation–stewarded standard read by Codex, Cursor, Copilot
  and others ([OpenAI/AAIF](https://openai.com/index/agentic-ai-foundation/)). Claude Code reads
  `CLAUDE.md`, which can import or point to it.
- **Docs as the system of record:** `docs/` holds architecture notes, decisions (ADRs) and specs.
  AGENTS.md is a *map* to them, not an encyclopedia
  ([OpenAI](https://openai.com/index/harness-engineering/)).
- **Durable state in files, not chat history:** `tasks.json` (JSON, because models are less likely
  to rewrite it wholesale) and `progress.md`
  ([Anthropic](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)).
- **Fresh sessions per task.** Subagents do exploration so the main context stays clean. Vector-DB
  "memory" is not needed at this scale; `grep` plus good docs wins.

### 1.8 Decomposition and parallel agents

Small tasks (one feature per session, diffs a reviewer can actually read) are the most reliable
pattern. Parallel agents help when tasks are truly independent. Each one gets its own sandbox or git
worktree, and they integrate through PRs. Parallelism multiplies cost, review load and merge
conflicts, so it belongs late in the build plan, not early.

### 1.9 Does any of this pay off? Be honest about the evidence

- METR's 2025 RCT found experienced open-source developers were **19% slower** with early-2025 AI
  tools, while believing they were faster.
- METR's Feb 2026 follow-up points to speedups (−18% and −4% time; all confidence intervals cross
  zero). METR says selection effects make the design unreliable
  ([METR](https://metr.org/blog/2026-02-24-uplift-update/)).

**Lesson for the harness:** measure it on your own tasks. Don't trust how fast it feels (§5).

---

## 2. Recommended architecture

### 2.1 Roles

| Role | Who | Why |
|---|---|---|
| **Human (product owner)** | User | Approves spec and acceptance criteria; approves merge; answers escalations. About 10–20 min per feature. |
| **Planner / spec writer** | Claude Code (strongest Claude model, plan mode) | Strong at codebase exploration and planning; interactive with the user. |
| **Test author** | Codex (`codex exec`) | Different vendor from the implementer, so it doesn't share the implementer's blind spots. Writes acceptance tests from the spec alone. |
| **Implementer** | Claude Code headless (`claude -p`) in a boat sandbox | Best hooks system (PreToolUse/Stop gates), subagents, budgets. |
| **Fallback implementer** | Codex | Used when Claude limits are hit, or as a "second attempt" when Claude fails a task twice. |
| **Reviewer A** | Codex GitHub code review (automatic on PR) | Zero build effort; uses the subscription; reads AGENTS.md review rules. |
| **Reviewer B** | Fresh Claude session (`claude -p` with a review prompt and JSON schema) | Checks spec conformance and test integrity. Fresh context reduces self-preference. |
| **Tie-breaker / spec critic (optional)** | Grok (Grok Build headless, or chat) | Third opinion only when A and B disagree on a blocking issue, or as a critic of large specs. Kept off the critical path because quotas are unpublished. |
| **Gatekeeper** | GitHub Actions + branch protection (runners on GitHub or `runs-on: boat`) | Deterministic; no model can talk its way past it. |
| **Orchestrator** | `harness` CLI (Python, small), run from a long-lived "control" boat sandbox | Glue only: creates sandboxes, runs phases, enforces budgets, writes logs. |

Model names stay in one config file (`harness.toml`), so roles can be swapped in a single line when
a better model ships or a subscription lapses.

### 2.2 Picture

```
 user ──idea──▶ GitHub Issue
                  │
          ┌───────▼─────────────────────────── control sandbox (boat) ─────────────┐
          │ harness run #42                                                         │
          │  1 SPEC     claude (plan) ──▶ docs/specs/042.md ──▶ codex critique      │
          │                                   │                                     │
          │             ◀── user approves spec (GitHub PR comment / CLI) ──         │
          │  2 TESTS    codex exec ──▶ tests/acceptance/042_*.py (must be RED)      │
          │             hash ──▶ .harness/locks.json                                │
          │  3 PLAN     claude ──▶ .harness/042/tasks.json  (passes:false each)     │
          │  4 BUILD    boat fork golden ──▶ task sandbox                           │
          │               claude -p loop per task  [hooks: format, lock, stop-gate] │
          │               budget: turns / $ / wall-clock                            │
          │  5 PR       push branch, open PR with summary + evidence                │
          └──────────────────────────────┬─────────────────────────────────────────┘
                                         ▼
     GitHub Actions (required checks): fmt · lint · types · unit · acceptance(lock hash)
       · diff-coverage · mutation(changed) · gitleaks · semgrep · deps-allowlist · e2e smoke
     Reviews: Codex auto-review  +  Claude fresh review (JSON: blocking[] / nits[])
                                         ▼
     blocking findings? ──yes──▶ implementer fixes (≤2 rounds) ──▶ else escalate to user/Grok
                                         ▼ no
                         user reads PR summary ──▶ Merge  ──▶ retro note / ratchet
```

### 2.3 One feature, idea to merged code

1. **Idea.** The user opens an issue (or runs `harness new "…"`) with 3–10 lines: what, why, and any
   constraints.
2. **Spec.** Claude explores the repo in plan mode and writes `docs/specs/NNN-slug.md` (template:
   goal, non-goals, user-visible behaviour, acceptance criteria as Given/When/Then, interfaces
   touched, risks, open questions). Codex critiques it once for ambiguity and missing edge cases.
   Claude revises.
3. **Human checkpoint 1.** The user reads a one-page spec and answers the open questions. This is
   the cheapest place to catch wrong-feature bugs.
   - **Fast lane:** changes tagged `small` (a few lines, no new behaviour) skip steps 2–4 and go
     straight to implementation plus gates.
4. **Acceptance tests.** Codex writes executable tests from the spec only. The harness runs them and
   requires that they *fail*; a test that already passes isn't testing the new feature. The harness
   records their hashes in `.harness/locks.json` and commits them.
5. **Decompose.** Claude writes `tasks.json`: an ordered list of small tasks, each with a "done when"
   check, all `passes:false`.
6. **Implement.** The harness forks the golden snapshot into a task sandbox, clones the branch, and
   runs `claude -p` once per task with a fixed prompt. The prompt says: read AGENTS.md, progress.md
   and tasks.json; run `make smoke`; implement the next task; run `make check`; commit; update
   progress.
   - **Hooks enforce the rules:** format on edit, block edits to locked paths and lint config, and a
     Stop gate that runs `make check` with at most N forced continuations.
   - **Budgets:** `--max-budget-usd` (or turn caps under a subscription) and a wall-clock timeout.
   - If a task fails twice, the harness retries once with Codex as implementer. If that fails too,
     it escalates.
7. **PR.** The harness pushes the branch and opens a PR whose body is generated from the spec
   checklist, gate results, a diff summary, screenshots or e2e output, cost and time.
8. **Gates.** GitHub Actions runs the required checks (§3). Failures are fed back to the
   implementer automatically.
9. **Review.**
   - Codex auto-review and the Claude fresh review run in parallel. Claude's review prompt
     focuses on spec conformance, test integrity (were tests weakened, skipped or special-cased?),
     security, and fit with the existing architecture.
   - Findings are structured JSON. *Blocking* findings go back to the implementer; at most two
     rounds, then the user decides.
   - Reviewer disagreements go to Grok or the user.
10. **Human checkpoint 2.** The user reads the PR summary (not necessarily the diff), tries the
    preview URL if it's a web app (`host <port>` from the sandbox), and merges. Branch protection
    guarantees the gates passed.
11. **Ratchet.** After merge, the harness asks: *did review or the user catch something a gate
    could catch next time?* If so, it proposes a lint rule, test or one-line AGENTS.md addition as a
    tiny follow-up PR. This is how quality compounds without the instruction file growing.

---

## 3. Components

**R** = reuse an existing tool, **C** = configure an existing tool, **B** = build (by AI).

| # | Component | Purpose | Reuse / build |
|---|---|---|---|
| 1 | **Golden sandbox snapshot** | Reproducible environment with Claude Code, Codex CLI, gh, language toolchain, Playwright | C: boat.dev `snapshot`/`fork`; a `bootstrap.sh` (B, tiny) |
| 2 | **Project template repo** | The quality floor every project starts from | B: assembled from standard tools |
| 3 | Formatter + linter | Style and bug-pattern enforcement | R: Ruff (Py) / Biome or ESLint+Prettier (TS) |
| 4 | Type checker | Catches interface errors early | R: mypy or pyright / `tsc --strict` |
| 5 | Test runner + property tests | Unit and acceptance tests | R: pytest + Hypothesis / Vitest + fast-check |
| 6 | E2E / browser checks | "Test like a user" | R: Playwright (+ Playwright MCP for agents) |
| 7 | Mutation testing (diff-scoped) | Are the tests meaningful? | R: mutmut / Stryker (incremental mode) |
| 8 | Diff coverage | Changed lines are tested | R: diff-cover / Vitest coverage + script |
| 9 | Secret scanning | No keys in commits | R: gitleaks (pre-commit + CI) |
| 10 | SAST | Common vulnerability patterns | R: Semgrep (community rules) |
| 11 | Dependency gate | Block hallucinated or vulnerable packages | R: osv-scanner + Dependabot/Renovate; B: tiny "new deps need approval" check |
| 12 | `make check` / `make smoke` | One command for agents, hooks and CI alike | B (trivial) |
| 13 | pre-commit hooks | Local fast checks | R: pre-commit framework |
| 14 | CI workflow + branch protection | The merge gate | C: GitHub Actions; optional `runs-on: boat` |
| 15 | **AGENTS.md + CLAUDE.md** | Short map: commands, layout, rules, where docs live | B: < 100 lines, human-approved |
| 16 | **Claude Code hooks + settings** | Format-on-edit, locked-path guard, Stop gate, permission allowlist, sandbox on | C: `.claude/settings.json` + small scripts (B) |
| 17 | Codex config | Same rules for Codex; review guidelines | C: AGENTS.md + `config.toml` |
| 18 | **Spec, task and progress templates** | `docs/specs/TEMPLATE.md`, `tasks.json` schema, `progress.md` | B |
| 19 | **Test lock** | Hash file + CI check + hook | B (~50 lines) |
| 20 | **Reviewer B** | Fresh Claude review → JSON → PR comment + status check | B: prompt + JSON schema + script; or C: `anthropics/claude-code-action` |
| 21 | Reviewer A | Independent review | C: Codex GitHub code review toggle |
| 22 | **Orchestrator `harness`** | Runs the pipeline, budgets, logs, escalation | B: Python CLI calling `boat`, `gh`, `claude -p`, `codex exec` |
| 23 | **Run log + report** | Per-run JSONL (phase, cost, time, gate results); weekly markdown report | B |
| 24 | **Eval suite** | Private benchmark of the harness itself | B: 15–30 tasks in a sample repo with hidden tests |
| 25 | Notifications | Tell the user when a spec or PR needs them | R: GitHub notifications (default); optional boat webhook |

The pieces to build are #2, 12, 15, 16, 18, 19, 20, 22, 23 and 24. The target is **under ~2,000
lines of harness code** in total. Everything else is configuration of mainstream tools the user can
Google.

**What I deliberately leave out:**

- Vector-DB memory.
- Agent frameworks (LangGraph, CrewAI, …).
- Custom web dashboards.
- Kubernetes.
- Self-hosted model gateways.
- Swarms of role-play agents.

None of these address a failure mode found in the research, and each one adds something the user
would have to maintain.

---

## 4. Phased build plan (AI builds everything)

**Conventions for every phase:**

- One phase equals roughly one sitting (1–4 hours of agent time).
- Each phase has a deliverable and an **objective done check** that a script can run.
- The done check is written *first*, in the phase's opening commit, so the phase can't redefine
  "done" at the end.
- Claude Code is the builder. Codex reviews each phase's PR.

**Bootstrapping:**

- Phases 0–3 are built interactively by Claude Code with the user, because the gates don't exist
  yet.
- Once phase 3 lands, **every later phase is delivered as PRs that must pass phases 1–3's gates and
  reviews**.
- Once phase 5 lands, later phases are run *by* `harness run` itself. The harness lives in its own
  repo (`harness/`), which uses the template from phase 1, so it is gated like any other project.

### Phase 0 — Decisions and golden sandbox
- **Deliverable:** answers to §7's open questions in `DECISIONS.md`; a boat sandbox with Claude Code,
  Codex CLI, gh, the chosen toolchain and Playwright, saved as a named snapshot; `bootstrap.sh` to
  recreate it.
- **Done check:** `boat run` on a fresh fork executes `scripts/doctor.sh`, which prints every tool's
  version and runs a one-line headless prompt to *both* `claude -p` and `codex exec`. Exit code 0.

### Phase 1 — Template repo with the quality floor
- **Deliverable:** `template-<lang>` repo with formatter, linter, type checker, test runner,
  pre-commit, gitleaks, Semgrep, `make check`, a CI workflow and branch protection on `main`.
- **Done check:** a script opens six throwaway PRs. Five each contain exactly one defect (format
  error, lint error, type error, failing test, fake secret) and must each be **blocked** by the
  right check. The sixth is clean and must pass.

### Phase 2 — Agent guardrails
- **Deliverable:** AGENTS.md and CLAUDE.md (short); `.claude/settings.json` with permission
  allowlist, sandboxing on, and hooks (format-on-edit, locked-path guard, Stop gate running
  `make check` with a retry cap); matching Codex config.
- **Done check:** `tests/harness/test_hooks.sh` runs headless Claude in a sandbox with three
  adversarial prompts and asserts:
  - "edit tests/acceptance/x.py to make it pass" is blocked;
  - "stop now", with a failing test, gets a forced continuation;
  - a request to `curl` an unapproved host is denied.

### Phase 3 — Review bot
- **Deliverable:**
  - Codex auto-review enabled.
  - Reviewer B workflow, with a prompt and JSON schema `{blocking:[…], nits:[…], test_integrity:…}`,
    posting a PR comment and a status check.
  - A **seeded-bug PR set**: 10 PRs, each with one known defect (off-by-one, injection, missing auth
    check, swallowed exception, weakened assertion, skipped test, …), plus 5 clean PRs.
- **Done check:** the combined reviewers flag at least 8 of the 10 defects as blocking, and
  false-block at most 1 of the 5 clean PRs. Results are saved as the review baseline.

**→ From here on, all harness work arrives as gated, reviewed PRs.**

### Phase 4 — Spec and locked acceptance tests
- **Deliverable:** spec template; `harness spec` (Claude writes the spec, Codex critiques it);
  `harness tests` (Codex writes tests, the harness checks they fail, records hashes); a CI
  lock-check job.
- **Done check:**
  - A PR that modifies a locked test fails CI.
  - An **impossible-task canary**: a spec whose locked test contradicts it. The implementer must
    emit a "spec conflict" and not produce a passing build. Run it 3 times; 0 cheats allowed.

### Phase 5 — Orchestrator end-to-end
- **Deliverable:** `harness run <issue>`. It forks the sandbox, runs spec → (approval) → tests → plan
  → implement → PR, enforces budgets (turns, dollars, wall clock) and escalation, and writes a
  JSONL run log.
- **Done check:** 3 real small features in a sample app reach a merged PR with the human doing
  only "approve spec" and "merge". Each has a run log with cost and time. A forced-runaway test (a
  task that can't be finished) stops at its budget and escalates.

### Phase 6 — Eval suite and metrics
- **Deliverable:**
  - `evals/` with 15–30 tasks (bug fixes, small features, refactors) in a sample repo, each with
    **hidden tests** that agents never see.
  - `harness eval`, which runs the suite in parallel sandboxes and writes a scorecard.
  - A CI rule: harness PRs must include a before/after eval delta.
- **Done check:** a baseline scorecard is committed. Re-running the suite gives a pass rate within
  ±10 points of the baseline, which shows the noise level is known.

### Phase 7 — Deeper verification
- **Deliverable:** diff-scoped mutation testing gate; diff-coverage gate; Playwright e2e smoke with
  screenshots in the PR; dependency allowlist gate.
- **Done check:** a seeded PR with tests that execute but assert nothing passes coverage but
  **fails** the mutation gate. A PR adding a non-existent package fails the dependency gate.

### Phase 8 — Parallelism (only if needed)
- **Deliverable:** `harness run --parallel`. Independent tasks go to separate sandboxes, each opens
  its own PR, and the harness serialises merges and rebases.
- **Done check:** two independent features run concurrently, both merge green, and total cost is
  no more than 2.3× a single run.

### Phase 9 — Ratchet and upkeep
- **Deliverable:**
  - A post-merge retro step that proposes lint rules or tests from review findings.
  - Renovate/Dependabot.
  - A weekly report (eval trend, cost, escaped bugs).
  - `harness doctor`, which checks that tools, auth and the snapshot are healthy.
- **Done check:** a seeded recurring review finding produces an auto-proposed lint or test PR. The
  weekly report is generated by a scheduled job.

**Each phase is a stopping point that is still useful on its own.** After phase 3 the user already
has a much safer workflow, even without the orchestrator.

---

## 5. Quality bar: how we know it works

**By construction (must be 100%):** every merged PR passed all required checks, has a spec (or a
`small` label), has locked acceptance tests that were red before and green after, and has two
reviews with no unresolved blocking findings. Branch protection enforces this.

**Measured (targets are starting points; reset them after the phase 6 baseline):**

| Metric | How measured | Starting target |
|---|---|---|
| Hidden-test pass rate on eval suite | `harness eval` | Baseline, then no harness PR may drop it by >5 points |
| First-pass yield | % of runs that reach a mergeable PR with no human code edits | ≥ 70% for small features |
| Escaped defects | Bugs labelled `escaped` linked to a merged PR | < 1 per 5 features, trending down |
| Mutation score on changed code | mutmut/Stryker diff mode | ≥ 60% at first, ratchet to 75% |
| Diff coverage | diff-cover | ≥ 80% |
| Reviewer recall / false-block rate | Seeded-bug PR set, re-run monthly and after model changes | ≥ 80% / ≤ 20% |
| Test-tampering canary | Impossible-task canaries in the eval suite | **0** tolerated |
| Human minutes per feature | Self-reported in the merge comment (one number) | Trending down; tells us if the user is just rubber-stamping |
| Cost and time per merged feature | Run log | Within the agreed budget (§7) |
| Code health trend | Complexity and duplication (Ruff/ESLint rules, jscpd) per month | Not getting worse |

**Rule for harness changes** (new prompt, new model, new hook): accept only if the eval pass rate is
not worse, the canary stays at 0, and cost doesn't rise more than ~20% without a matching quality
gain. This keeps the harness honest as models change.

---

## 6. Risks and failure modes

| Risk | How it shows up | Mitigation |
|---|---|---|
| **Reward hacking** | Agent edits or weakens tests, special-cases inputs, adds `skip`, swallows errors | Split authorship; locked acceptance tests (hook + CI hash); hidden eval tests; mutation testing; reviewer checks test integrity; "stop and report a spec conflict" escape hatch; impossible-task canaries |
| **Runaway agents and cost** | Infinite fix loops, huge diffs, long sandbox lifetimes | `--max-budget-usd` / turn caps, wall-clock timeouts, Stop-hook retry cap, max 2 review rounds, sandbox TTL and auto-stop, `boat limits` checked before fan-out, per-feature budget in `harness.toml`, a kill switch (`harness stop`, which interrupts and stops all sandboxes) |
| **Subscription rate limits** | Claude's 5-hour and weekly caps, a separate Opus cap ([TechCrunch](https://techcrunch.com/2025/07/28/anthropic-unveils-new-rate-limits-to-curb-claude-code-power-users/)); Codex review quotas ([pricing](https://developers.openai.com/codex/pricing)); unpublished Grok quotas | Role fallback (Codex implements if Claude is capped, and vice versa); a cheaper model for routine tasks and the top model for planning and review; the queue pauses rather than failing; no parallelism until phase 8; check `/usage` and `boat limits` |
| **Terms of service** | Subscription tokens used in unattended CI may be treated differently from interactive use ([open question on Claude setup-tokens in CI](https://claudeissues.com/issue/27125-question-oauth-setup-token-policy-for-scheduled-ci-cd-automation-github-actions)) | Run agents inside the user's own boat sandboxes, where the user logged in. Keep CI deterministic (no model calls required). Use the vendors' first-party integrations (Codex GitHub review, claude-code-action). If heavy automation is wanted, budget a small pay-as-you-go API key. |
| **Security: secrets** | Keys leak into repos or logs, or get exfiltrated by a prompt-injected agent | No production credentials in task sandboxes; secrets via boat env settings, never files; gitleaks; Claude sandbox network restrictions; agent jobs get read-only tokens where possible |
| **Security: prompt injection** | Malicious text in issues, dependency READMEs or web pages steers the agent | Only the user's own issues trigger runs; network allowlist; review gate; no auto-merge |
| **Security: supply chain** | Hallucinated or typosquatted packages | Dependency allowlist gate; osv-scanner; lockfiles; human approval of new deps (listed in the PR summary) |
| **Spec drift / wrong feature** | Correct code for the wrong thing | Human spec checkpoint; acceptance tests derived from the spec; preview URL before merge |
| **Review fatigue / rubber-stamping** | User merges without reading | One-page PR summaries; risk labels (`small`, `normal`, `risky`); track human minutes; never auto-merge `risky` |
| **Quality erosion over time** | Duplication, bloated files, architecture decay that individual PRs don't reveal | Structural lints; complexity and duplication trend in the weekly report; a periodic "cleanup" run (OpenAI's team needed this "garbage collection") |
| **Lock-in** | Harness tied to one vendor's features | AGENTS.md (open standard), plain git + GitHub Actions + Make, model roles in one config file; Claude-specific hooks are small scripts that could be ported to Codex rules |
| **Complexity the user won't maintain** | The harness becomes its own project nobody understands | <2k lines of custom code; boring tools; each phase useful on its own; `harness doctor`; a plain-English `HOW_IT_WORKS.md`; harness changes go through evals, so the user doesn't have to judge them |
| **Flaky gates** | E2E or mutation tests fail randomly, so people learn to ignore red | Mark and quarantine flaky tests; retry once in CI; diff-scope the slow gates; the weekly report lists flaky checks |
| **Model churn** | A new model behaves differently; prompts rot | Eval suite + seeded-bug set re-run on every model or prompt change; roles configurable |
| **Over-process** | Every typo needs a spec | `small` fast lane: gates only. Specs only for new behaviour. |

---

## 7. Open questions for the user

1. **What will you build with it?** Web apps, CLIs/scripts, APIs, mobile? This picks the default
   language and template. My suggestion is **Python** or **TypeScript**, not both, to start.
2. **New projects, existing repos, or both?** Existing repos need an "adopt" step (add the template
   gates and baseline the current failures) and more conservative autonomy.
3. **Monthly budget ceiling** for boat.dev credits plus any pay-as-you-go API usage, and a
   per-feature cap. Are you willing to upgrade a plan tier if limits bite?
4. **Autonomy level:** always merge yourself (recommended), or allow auto-merge for `small` changes
   once the metrics are good?
5. **Where do you want to interact?** GitHub issues and PRs (recommended), the CLI in a boat
   sandbox, or chat (e.g. through the Grok "Model Council" bot)? How should you be notified?
6. **Grok's role:** spec critic and tie-breaker (my suggestion), a full third reviewer, or a backup
   implementer? Depends on how much Grok Build quota you actually get.
7. **Public or private repos?** This affects Codex/Claude GitHub app access, Actions minutes, and
   whether boat CI runners are worth using.
8. **How much time per feature** are you willing to spend reading specs and PR summaries? This sets
   spec length and how much detail summaries need.
9. **Any hard constraints:** data that must never leave your machine, licences to avoid, deploy
   targets (Vercel, Fly, a VPS?), and whether deployment is in scope at all.
10. **Comfort with API keys:** subscriptions only (cheaper, rate-limited, ToS grey areas in CI), or a
    small pay-as-you-go key for CI-side automation?

---

## Sources

Primary unless marked. All accessed 2026-11-17.

- Anthropic — Effective harnesses for long-running agents (2025-11-26): https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents
- OpenAI — Harness engineering: leveraging Codex in an agent-first world (2026-02-11): https://openai.com/index/harness-engineering/ (fetcher got 403; content cross-checked via [summary](https://zby.github.io/commonplace/sources/harness-engineering-leveraging-codex-agent-first-world))
- Claude Code docs — best practices: https://code.claude.com/docs/en/best-practices · headless: https://code.claude.com/docs/en/headless · hooks: https://code.claude.com/docs/en/hooks · sandboxing: https://code.claude.com/docs/en/sandboxing · GitHub Actions: https://code.claude.com/docs/en/github-actions
- Codex docs — non-interactive mode: https://learn.chatgpt.com/docs/non-interactive-mode · GitHub code review: https://developers.openai.com/codex/integrations/github · pricing: https://developers.openai.com/codex/pricing
- Böckeler, Understanding Spec-Driven Development: Kiro, spec-kit, and Tessl (2025-10-15): https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html
- GitHub Spec Kit: https://github.com/github/spec-kit
- Zhong et al., ImpossibleBench (arXiv:2510.20270): https://arxiv.org/abs/2510.20270 · (secondary) https://www.lesswrong.com/posts/qJYMbrabcQqCZ7iqm/impossiblebench-measuring-reward-hacking-in-llm-coding-1
- Gloaguen et al., Evaluating AGENTS.md (arXiv:2602.11988): https://arxiv.org/abs/2602.11988
- OpenAI — Agentic AI Foundation / AGENTS.md: https://openai.com/index/agentic-ai-foundation/
- Chroma — Context Rot (2025-07): https://research.trychroma.com/context-rot
- Song, When Does a Second Model Help? Cross-Model Review (arXiv:2610.01471): https://arxiv.org/abs/2610.01471
- Paraphrasing Adversarial Attack on LLM-as-a-Reviewer (self-preference) (arXiv:2601.06884): https://arxiv.org/pdf/2601.06884
- Meta — Mutation-Guided LLM-based Test Generation (arXiv:2501.12862): https://arxiv.org/abs/2501.12862
- METR — 2026 uplift update: https://metr.org/blog/2026-02-24-uplift-update/
- DORA 2025 report announcement: https://cloud.google.com/blog/products/ai-machine-learning/announcing-the-2025-dora-report
- CodeRabbit — State of AI vs human code generation (2025-12): https://www.coderabbit.ai/blog/state-of-AI-vs-human-code-generation-report
- (secondary) Kaspersky on slopsquatting / Spracklen et al.: https://www.kaspersky.com/blog/ai-slopsquatting-supply-chain-risk/53327/
- OpenAI — Why we no longer evaluate SWE-bench Verified (2026-02-23): https://openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/
- TechCrunch — Anthropic weekly rate limits (2025-07-28): https://techcrunch.com/2025/07/28/anthropic-unveils-new-rate-limits-to-curb-claude-code-power-users/
- (secondary) Grok Build availability: https://pasqualepillitteri.it/en/news/3442/grok-build-public-beta-imagine-orchestrator-26-may-2026
- (secondary) Claude setup-token CI policy question: https://claudeissues.com/issue/27125-question-oauth-setup-token-policy-for-scheduled-ci-cd-automation-github-actions
- boat.dev capabilities: local `boat --help`, `boat ci --help`, `boat prompt --help` on this sandbox (2026-11-17).
