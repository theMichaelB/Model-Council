# Codex proposal: a small, evidence-driven coding harness

**Independent council research · 17 November 2026**

## Executive summary

Build a **thin, deterministic controller around Claude Code, Codex, GitHub, and disposable boat.dev workers**. Do not build another coding agent, a permanent model parliament, or a complex agent framework. The agents already supply the tool-use loops; the missing product is reliable task intake, bounded execution, independent verification, recovery, and a clear approval screen.

Claude should initially plan and implement; Codex should independently review and verify. Swap those roles in evaluations rather than assuming a permanent winner. Grok should challenge consequential specifications when useful, but must not become a required dependency without verified automation access and billing. Start sequentially; add at most two parallel implementation workers only after measurement justifies them.

**AI writes all implementation, tests, documentation, and deployment scripts.** The user supplies preferences, signs into services, authorizes spending, and approves product/security decisions—not code. The deliverable is a working harness with a one-command setup, GitHub-based controls, evidence-backed PRs, budget limits, and recovery instructions. It can improve reliability substantially; no credible architecture guarantees correct software from arbitrary requests.

## 1. State of the art: useful patterns, not magic

### What the recent evidence supports

An effective agent repeatedly inspects context, chooses actions, executes tools, and observes results. Long tasks fail when that loop loses state, expands scope, or accepts its own claims as proof. Anthropic's November 2025 harness demonstrated initialization, incremental features, persistent progress records, and browser-based verification across sessions. These are useful engineering patterns, not proof of universal production readiness. [Anthropic: long-running agents](https://www.anthropic.com/research/effective-harnesses-for-long-running-agents).

The stronger 2026 lesson is **adapt the harness to the task and model**. Anthropic's March application-development experiments retained planning and evaluation but simplified orchestration as capabilities improved. Its evaluator initially approved real defects and required calibration; for easier tasks, evaluation became unnecessary overhead. Thus “more agents” and “more iterations” are not monotonic improvements. [Anthropic: harness design](https://www.anthropic.com/research/harness-design-long-running-apps).

OpenAI's February 2026 internal case study emphasizes a legible repository, short instruction files pointing into maintained documentation, and mechanical architectural constraints. It supports making the environment easier to navigate rather than accumulating an enormous prompt. This is an experienced team's case study, not evidence that a casual coder can reproduce its reported productivity simply by installing a framework. [OpenAI: harness engineering](https://openai.com/index/harness-engineering/).

### Practices to adopt

- **Spec-first, proportionately:** before implementation, record observable outcomes, exclusions, examples, error cases, and risks. A small bug needs a short contract, not a planning ceremony. Large features become vertical slices with explicit dependencies.
- **Test-first where behavior is clear:** reproduce bugs with failing tests; specify important acceptance tests before implementation. Keep unit, integration, and actual user-journey tests. Tests generated after seeing the implementation are useful, but weaker evidence of independent correctness.
- **Mechanical checks:** formatting, linting, type checking, builds, dependency/secret checks, and CI are mandatory machine work, not model judgments. Existing projects retain their tooling; do not migrate stacks to please the harness.
- **Independent verification:** a second fresh session receives the approved spec and candidate commit, not the implementer's persuasive narrative. It searches for counterexamples and executes the application. Model diversity may expose different mistakes; agreement is not a correctness proof.
- **Targeted memory:** short `AGENTS.md`, a compatible `CLAUDE.md` entry point, architecture notes, task contracts, and verified command recipes. Retrieve with ordinary file search first. Versioned evidence is memory; an unbounded transcript is not. Context selection and just-in-time retrieval are established design concerns. [Anthropic: context engineering](https://www.anthropic.com/research/effective-context-engineering-for-ai-agents).
- **Limited parallelism:** delegate independent discovery or separate modules with clear interfaces. Anthropic's multi-agent research work concerns separable research and reports meaningful coordination/resource costs; its results should not be extrapolated into a coding-team multiplier. [Anthropic: multi-agent research](https://www.anthropic.com/research/multi-agent-research-system).

### What not to believe

Neither green tests nor a benchmark score establish product quality. In February 2026, OpenAI reported test-design and contamination problems in SWE-bench Verified and stopped using it for frontier comparisons. Use public benchmarks for orientation, not as this user's acceptance contract. [OpenAI: benchmark limitations](https://openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/).

Likewise, productivity must be measured locally. METR's early-2025 randomized study found a 19% slowdown for experienced maintainers on their repositories. Its February 2026 update explains why selection effects complicate newer estimates and suggests improvements may have occurred. Neither establishes today's outcome for this casual coder. Measure completed, accepted features and user effort—not generated code or perceived speed. [2025 paper](https://arxiv.org/abs/2507.09089), [2026 update](https://metr.org/blog/2026-02-24-uplift-update/).

## 2. Recommended architecture

### A deliberately small system

```text
User ↔ GitHub issue / approval comment
                 ↓
Trusted controller: state machine + SQLite ledger + policy/budget
                 ↓
Disposable boat implementation VM → candidate commit/patch
                 ↓
Separate verification VM + trusted CI + independent review
                 ↓
Evidence PR → user approval → restricted merge service → main
```

**Controller:** a small TypeScript program, SQLite, JSON Schema validation, and a process supervisor. No LLM chooses whether a gate passed or grants itself additional permission. States are `intake → planned → approved → implementing → verifying → review → ready → merged`, with explicit `blocked`, `failed`, and `cancelled` outcomes. Each transition records task ID, base/candidate SHA, policy version, artifacts, and timestamps. Model output is untrusted input validated against schemas.

**Execution:** use Boat's supported TypeScript SDK/API behind one adapter; use CLI commands during bootstrap and troubleshooting. Its documented interfaces cover sandbox lifecycle, command execution, agent runs, events, and snapshots. Save explicit sandbox IDs; never rely on shell-local `current` in durable jobs. [Boat SDK/API](https://docs.boat.dev/sdks/overview).

Maintain a clean, secret-free template with pinned tools. Create implementation and verification workers separately. Never fork the user's personal logged-in HOME as a generic template. A stopped VM/snapshot is recovery material, not the sole backup: Boat preserves disk, not running processes, and stopped storage has retention limits. Store commits and redacted evidence outside the VM. [Boat lifecycle overview](https://boat.dev/).

**Security boundary:** workers receive no Boat management credential, merge authority, production secret, or policy-repository write access. The controller imports a patch rather than giving workers GitHub write credentials. Enforce permissions outside the prompt: OS users/containers, writable paths, restricted credential delivery, and deny-by-default network access where feasible. CLI login credentials remain sensitive even in a disposable VM; test whether the chosen credential mechanism exposes them to arbitrary child commands. If adequate isolation cannot be demonstrated, restrict the prototype to trusted private code and synthetic data, with that limitation explicit.

### Concrete model/tool allocation and subscription caveats

| Role | Initial choice | Boundary |
|---|---|---|
| Requirements and plan | Claude Code | User approves meaning and scope |
| Implementation and bounded repair | Claude Code native CLI | Writes only task workspace; cannot merge |
| Independent code review and QA planning | Codex native CLI, fresh session | Produces findings, not final authority |
| Actual verification | Test tools and clean worker | Controller checks exit codes and artifacts |
| Occasional adversarial spec critique | Grok | Optional; no unattended UI scraping |
| Scheduling, gates, merge | Ordinary program + GitHub | Never delegated to model discretion |

Codex documents `codex exec`, event/structured outputs, and saved authentication. Claude documents programmatic CLI execution with `claude -p`, structured output, and tool restrictions. Pin tested versions; use structured events rather than parsing conversational prose. [Codex automation](https://learn.chatgpt.com/docs/non-interactive-mode), [Claude automation](https://code.claude.com/docs/en/headless).

**Do not equate subscriptions with API credit or permission.** Start by checking each actual account and testing the permitted native CLI path. Anthropic distinguishes ordinary native subscription use from product/Agent SDK integration, for which its guidance calls for API authentication; do not build an SDK service that routes subscription OAuth tokens. If this personal controller's intended usage is not clearly covered, obtain clarification or use an explicitly budgeted API path. Codex account-auth automation also has security constraints; never export account credentials into public-repository CI. [Claude credential policy](https://code.claude.com/docs/en/legal-and-compliance), [Codex automation/auth guidance](https://learn.chatgpt.com/docs/non-interactive-mode).

Grok offerings have changed: current documentation discusses shared first-party subscription usage, while the user's exact plan and API entitlement are unspecified. Verify account-specific access, supported automation, and marginal cost; do not assume either unlimited API access or that older billing rules still apply. Initially generate a sanitized critique packet for optional manual submission; automate only through a supported, authorized interface. The core works without Grok. [xAI subscription overview](https://docs.x.ai/x-subscription/overview), [Grok usage FAQ](https://docs.x.ai/grok/faq).

### One feature, from idea to merge

1. **Intake:** user requests “export filtered records to CSV.” Planner inspects the repository and asks only material questions: permissions, fields, scale, privacy, and desired behavior.
2. **Contract:** define required columns, escaping, empty results, Unicode, unauthorized access, and large exports. Record non-goals and a risk tier. User approves the short behavioral summary.
3. **Independent test design:** verifier drafts acceptance scenarios before viewing implementation; controller freezes a versioned suite outside implementation-worker write access. Public criteria remain visible; selected test data can remain held out.
4. **Implement:** Claude gets the contract, base SHA, relevant docs, command recipes, and a budget. It adds focused tests, implements, and runs local checks. It returns a candidate patch and unresolved issues.
5. **Verify:** controller checks protected-path changes, applies the patch to a clean checkout, installs locked dependencies, and runs trusted commands. Test the running app, real test database, downloads, and permissions—not merely mocked return values. Playwright recommends verifying user-visible behavior and isolated tests. [Playwright practices](https://playwright.dev/docs/best-practices).
6. **Review:** Codex examines code and requirement coverage. Findings need severity, location, reproduction or reasoning, and a suggested check. No majority voting. Unresolved correctness/security disagreements block or go to the user.
7. **Repair:** at most two repair cycles, rerunning all affected gates. Then stop with a useful failure report; never silently relax the contract.
8. **PR:** present a plain-language summary, acceptance-to-evidence table, screenshots/traces where relevant, costs, remaining risks, and rollback instructions. The user can try a preview with synthetic data.
9. **Merge:** require current-SHA checks and explicit approval initially. A separate narrowly authorized service merges only the verified candidate. Rebase or any new commit invalidates previous evidence. Production deployment is a separate authorization boundary.

## 3. Components: reuse first, build only the glue

| Component | Purpose | Reuse / AI-built addition |
|---|---|---|
| Intake and approvals | Familiar user interface | GitHub Issues/PRs; small contract template and authenticated approval parser |
| Agent execution | Existing coding loops | Claude Code and Codex; thin adapters and schema validation |
| Isolation and recovery | Disposable compute | Boat SDK, clean templates, snapshots; worker lifecycle wrapper |
| Version control | Safe integration | Git branches, patches, GitHub; candidate-SHA/evidence linkage |
| Controller | Bound autonomy | Build small state machine, SQLite ledger, leases, retries, cancellation |
| Project commands | Reproducible checks | Existing tools; generate one `check` command per repository |
| Default new web-project checks | Boring initial stack | TypeScript strict checks, ESLint, Prettier, Vitest, Playwright; use pytest/Ruff equivalents for Python |
| Acceptance verification | Detect missing/wrong behavior | Standard test runners; protected evaluator suite and evidence manifest |
| Security checks | Detect preventable exposure | Dependency audit, Gitleaks, selected static-analysis rules; triage report |
| Merge gates | Enforcement | GitHub branch rules and Actions; trusted aggregate check/merge identity |
| Durable context | Avoid rediscovery and drift | Markdown, Git, file search; command validation and lightweight doc checks |
| Quality evaluation | Test the harness itself | Existing runners; small task corpus and fault-injection fixtures |
| Operations | No maintenance puzzle | Supervisor, logs, backups; `doctor`, `status`, `cancel`, recovery runbook |

Do **not** initially add LangGraph/Temporal, a vector database, a custom IDE/dashboard, a message broker, or autonomous model-selection machinery. Revisit only when a demonstrated failure requires them. TypeScript is a proposed default for the harness, not a mandate for every project.

## 4. Phased build plan: AI builds everything

The only unavoidable human work is account login, consent, product decisions, and trying the result. AI supplies scripts and guided actions for setup; the user should not have to assemble a framework.

| Phase | AI-created deliverable | Objective done check |
|---|---|---|
| **0. Capability and budget probe** | `doctor` report: installed versions, native headless access, GitHub permissions/plan, Boat lifecycle, billing mode; no secrets in report | One permitted tiny run per required provider; synthetic worker created/stopped; failure and cancellation tested; user accepts budget/auth path |
| **1. Project safety template** | Sample app, pinned dependencies, one check command, CI, short instructions, install/uninstall guide | Fresh VM installs and passes; deliberate type error, failing test, and disabled check are rejected; branch protection verified where supported |
| **2. Sequential runner** | Intake → approved contract → one worker → verified patch → PR; durable ledger, timeouts, cancel | Three small fixture tasks produce runnable PRs; killed run resumes without duplicate PR/VM; malformed output, quota denial, and timeout fail closed |
| **3. Independent verification** | Separate Codex review, protected acceptance suite, current-SHA evidence bundle | Seeded missing feature, broken persistence, permission flaw, test deletion, and stale evidence each block merge; valid candidate passes |
| **4. Pilot and hardening** | Task evaluations, budget accounting, backup/restore, user-facing status and recovery | Quality targets below met; restore on a new VM demonstrated; 20 pilot tasks completed with recorded user effort and failure reasons |
| **5. Optional expansion** | At most two disjoint workers, selected Grok critique, low-risk auto-merge | Paired evaluations show useful improvement per unit cost; conflict, quota, cancellation, and leaked-resource tests pass; user explicitly opts in |

**Bootstrap:** phases 0–1 are ordinary AI-authored PRs produced with today's native agents and manually approved. Phase 2 then builds phase 3; phase 3 verifies phase 4. Keep a known-good tagged runner. The harness may propose changes to itself, but cannot change its active policy, evaluator, credentials, or merge rules mid-run. Harness changes run candidate-versus-baseline evaluations and require separate approval. Self-improvement means reviewed software updates, not self-authorized privilege expansion.

Do not advance because a demonstration looks impressive. Every phase ships its tests, documentation, rollback, and measured evidence. If phase 2 alone saves effort, it is a legitimate usable stopping point.

## 5. Quality bar: functional, maintainable, demonstrably bounded

### Per-change gates

Mandatory: clean install/build; formatter/linter/type checker; existing regression tests; contract-linked acceptance tests; independent findings resolved; protected configuration unchanged without authorization; no untriaged new high-severity security findings; current candidate SHA verified. Critical flows require failure-path and integration coverage. A migration requires rehearsal against representative synthetic data and a recovery plan.

The trusted aggregate gate must confirm expected jobs **executed successfully**, test counts are plausible, and required artifacts exist. GitHub can accept skipped/neutral required checks, and protection availability depends on repository/account plan. Bind status sources where supported, disallow worker bypass, and test the actual enforcement; never equate “workflow exists” with “gate enforced.” [GitHub branch protection](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches).

Maintainability review asks: does the patch fit existing architecture, avoid unnecessary dependencies, explain nonobvious decisions, and remain understandable? Visual quality and product usefulness still require user-calibrated judgment; don't reduce them to a model's numerical score.

### Harness evaluation, separate from project tests

Create 20 representative tasks: bug fixes, small features, refactors, dependency upgrades, and ambiguous requests that should trigger clarification. Include realistic permission, persistence, and error-handling failures. Keep a development set and a sealed holdout set; freeze the grader before evaluating candidates. Another AI authors and challenges the grader; the user approves plain-language expected outcomes. AI-generated tests are not automatically trustworthy.

Compare the harness against a single native-agent baseline on the same task snapshots and equal budgets. Record actual resolved model IDs, prompts, tool versions, elapsed time, observed usage, and unknown costs. Repeat a small subset three times to expose variability. Outcomes matter more than claimed success, and multiple trials and mixed graders are standard agent-evaluation techniques. [Anthropic: agent evals](https://www.anthropic.com/research/demystifying-evals-for-ai-agents).

**Provisional launch targets—not research-established guarantees:**

- At least 18/20 representative tasks accepted within two repair cycles; show task-level results and uncertainty, not just the percentage.
- Zero false “ready” outcomes on the seeded safety/gate-failure suite; 100% rejection of known unauthorized merge/policy actions.
- No orphan workers after cancellation/recovery tests; every run ends with an accounted terminal state.
- No more than ten minutes median user handling per routine task during the pilot, excluding the user's optional exploration; report total latency separately.
- Cost per accepted task below the user's agreed cap; no silent paid fallback. Track rework and defects over the following 30 days.

These are launch screens, not statistical proof of 90% real-world reliability. Continue collecting outcomes; report escaped defects and rollbacks, and add regressions after each incident.

For important business logic, run targeted mutation testing: deliberately alter behavior and check whether tests detect it. Stryker distinguishes killed mutants from survivors and uncovered code. Start by investigating every meaningful survivor in critical changed logic; do not demand an arbitrary universal mutation or coverage percentage. Mutation testing is expensive and complements, rather than replaces, acceptance testing. [Stryker metrics](https://stryker-mutator.io/docs/mutation-testing-elements/mutant-states-and-metrics/).

## 6. Risks and failure modes

| Risk | Mitigation |
|---|---|
| Runaway loops and VM bills | Start with one task/worker; proposed 30-minute agent invocation and 90-minute task limits, two repairs, worker TTL, daily cap, independent cleanup watchdog; user sets monetary caps before enablement |
| Subscription quotas or changing terms | Capability probe; backoff and stop; preserve state; no account rotation or automatic paid fallback; supported APIs only when authorized |
| Test reward hacking | Freeze external acceptance suite; reject deletions/skip additions/config weakening; verify actual behavior and artifacts; fresh checker cannot inherit implementer's claims |
| Shared blind spots | Different-model review, held-out cases, deterministic tools, mutation checks, and user-calibrated outcomes; no “two models agree” shortcut |
| Prompt injection and exfiltration | Treat repository text, issues, dependencies, and tool output as untrusted; minimal tools/egress; no production credentials; never trust instructions alone as isolation |
| CI as attack surface | Run untrusted candidate code without agent/merge secrets; protect verifier and workflow definitions; pin trusted workflows; merge credential used only outside code execution |
| Snapshot credential cloning | Secret-free templates, explicit provisioning, retention review, credential rotation after suspected exposure; Git and external backups remain authoritative |
| Controller crash or duplicate execution | SQLite transactions, task lease, idempotency keys, persisted VM IDs, startup reconciliation, latest-SHA checks before merge |
| Integration conflicts | Disjoint ownership, interface contracts, serial integration, full checks after combining; halt instead of repeatedly rebasing blindly |
| Complexity and lock-in | Thin provider/Boat adapters, portable Git/artifacts, one runtime/database, no custom UI; pinned releases plus AI-authored upgrade PRs validated by evals |
| User becomes the unpaid maintainer | One-command diagnostics, actionable failure summaries, scheduled bounded maintenance PRs, documented restore; no autonomous dependency churn |

Budget enforcement must distinguish VM spend, provider-billed usage, and subscription quota. If exact token or quota telemetry is unavailable, label it unknown and enforce time/concurrency limits; do not fabricate dollar precision. Rate limits are a reason to pause, not weaken verification.

## 7. Decisions only the user can make

1. Which real repository and three representative features should define the first pilot? Which languages, deployment targets, and data sensitivity apply?
2. What monthly incremental spend and per-task cap are acceptable? Is any API spending permitted, or must we remain subscription-only?
3. Which approvals should remain mandatory: specification, merge, migrations, production deployment? Recommendation: all consequential actions initially.
4. What does “high quality” mean here: design polish, accessibility, performance, security, maintainability—and what matters most when they conflict?
5. May code/specifications be sent to all three providers? Are there licensing, confidentiality, region, or retention restrictions?
6. Which exact subscription tiers and GitHub plan are available, and is the user willing to change plans if enforced private-repository gates require it?
7. What is the acceptable tradeoff between waiting for better verification and getting an earlier preview? Who receives a blocked-task notification?

**Recommendation:** approve phases 0–2 for one low-risk repository, then make further investment contingent on measured accepted output and reduced user effort. The right harness is the smallest one that repeatedly earns trust through evidence.
