# RFC: Agentic Inner-Loop Verification

**Status:** Proposed — design-partner validation requested  
**Date:** November 15, 2026  
**Owner:** CI / Developer Infrastructure (proposed)  
**Scope:** Local-first verification for coding agents; pre-PR developer workflow  
**Decision requested:** Approve discovery and a limited design-partner pilot, not a broad organizational mandate.

## 1. Executive summary

We propose a **local-first, agent-accessible verification platform** that brings useful CI checks into the coding agent's development loop—before a pull request exists. The platform would discover relevant checks, verify the local environment is ready, execute a targeted set of checks, provide concise remediation guidance, and produce a **behavior-oriented verification artifact** for reviewers.

The objective is **not** to reproduce the entire CI pipeline on every edit or to require a particular Claude Code plugin. It is to make verification **fast, relevant, actionable, and evidence-producing** while preserving authoritative, centrally enforced CI after PR creation.

The proposal has five interconnected capabilities:

1. **Intelligent check selection:** Identify checks applicable to the changed code and intended behavior.
2. **Targeted local execution:** Run checks efficiently with existing project tools, incremental execution, caching, and bounded concurrency.
3. **Environment readiness and reliability:** Detect missing prerequisites early; distinguish code failures from environment failures and inconclusive results.
4. **Direct, next-step feedback:** Return a short diagnosis, supporting evidence, and the next safe action instead of raw logs alone.
5. **Behavioral verification artifact:** Group new and existing tests by intended outcome, explain what their assertions exercise, and disclose coverage gaps.

**Recommendation:** Start with 2–3 backend design-partner teams. Ship a thin CLI plus optional Claude Code skills/hooks. Expand only after showing meaningful improvements over agents executing checks unaided.

## 2. Problem and opportunity

Today, engineers and coding agents can already run linters, builds, and tests locally. The gap is less about command availability and more about **consistent selection, timing, environment setup, interpretation, and evidence**:

- Agents must discover repository-specific commands and decide which are relevant, often with incomplete context.
- Integration tests may fail because of missing services, credentials, or incompatible runtimes; agents can mistake these for application defects.
- Full test suites are often too expensive to run after every edit; ad hoc selection risks missing regressions.
- A passing CI report lists checks and outcomes but rarely explains **which intended behaviors the tests substantiate**.
- Different teams use their own Claude Code skills, hooks, and workflows; an opinionated mandatory plugin introduces adoption friction.
- CI failures discovered only after PR creation lead to additional repair cycles and reviewer interruption.

The central hypothesis is that a **reliable, local verification contract**—rather than merely more test executions—will let agents resolve more actionable failures earlier and hand reviewers stronger evidence with less cognitive load.

### 2.1 Why now

Coding agents can modify multiple files, run commands, and respond to feedback in a single session. This increases the value of short feedback loops, but also increases the cost of ambiguous failures and unnecessary reruns. Existing industrial approaches to presubmit testing and test selection provide useful foundations; this RFC adapts them to **repeated verification of an evolving, potentially uncommitted workspace**.

## 3. Goals, non-goals, and principles

### Goals

- Make repository-approved verification discoverable through one stable local interface, callable by humans or agents.
- Catch and resolve actionable failures **before PR creation** without imposing a large increase in total task latency.
- Deliver check-specific preflight and classified failure outcomes, including safe remediation guidance.
- Produce compact, source-linked evidence that tells reviewers **what expected behavior was exercised and how**.
- Allow a CI team to define default pre-commit, pre-push, and agent-completion behavior while preserving team extensibility.
- Demonstrate voluntary repeat usage and measurable impact in a design-partner pilot before broad rollout.

### Non-goals (V1)

- Replacing authoritative PR CI, security attestations, compliance gates, or organization-wide required checks.
- Running the **entire** CI suite continuously or promising complete local/CI environment parity.
- Building remote execution infrastructure, ephemeral cloud runners, or a new test framework.
- Automatically writing tests or proving complete specification correctness.
- Forcing all engineers to adopt one Claude Code plugin or prohibiting custom skills.
- Making an LLM the source of truth for pass/fail outcomes or allowing unattended arbitrary remediation commands.

### Design principles

**Standardize verification capabilities, not agent behavior.** A versioned CLI and result contract are the platform; Claude Code plugins are optional adapters. **Make cheap checks frequent and expensive checks deliberate.** **Report evidence, not confidence theater.** **Preserve uncertainty.** A blocked or unselected check is never presented as passed. **Maintain a no-dead-end outcome:** each result contains a next action or an explicit escalation/blocker.

## 4. Proposed developer workflow

| Stage | Trigger | Platform behavior | Developer/agent outcome |
| --- | --- | --- | --- |
| Session start | Optional agent initialization or `verify doctor` | Discover repository checks; perform low-cost environment preflight; flag gaps | Know what can be verified before coding begins |
| During development | Explicit `verify run --profile fast`, or optional agent hook | Run lightweight relevant lint, type, build, or unit checks | Immediate actionable feedback |
| Task completion | Agent stop/task-complete hook | Select and execute affected checks; provide retry or remediation | Agent repairs before claiming completion |
| Pre-commit | Git hook, typically blocking for fast checks | Validate **staged content** where supported | Avoid committing obvious defects |
| Pre-push | Git hook, selective checks within a budget | Validate changes being pushed | Reduce preventable PR CI failures |
| Pre-PR | CLI/agent invocation | Run locally eligible readiness suite; construct behavioral evidence | PR with concise verification summary and disclosed gaps |
| Post-PR | Existing CI | Full required checks, policy and security gates, trusted attestations | Authoritative merge decision remains unchanged |

Default hooks should be configurable and resource-bounded. **Session-start preflight is advisory**, and unrelated missing dependencies must not block development. Check-specific preflight runs just before a selected check, with freshness-sensitive probes when needed.

## 5. System architecture and ownership

```text
Claude Code / Cursor / Codex / custom skills / developer terminal
            |  optional hooks, CLI, MCP adapter
            v
     Versioned `verify` CLI + result schema
            |
            v
     CI-owned verification coordinator
     +----------------------------------+
     | check catalog & change analysis  |
     | selection policy & lifecycle     |
     | targeted readiness / preflight   |
     | local execution & caching        |
     | failure parsing / remediation    |
     | evidence & report assembly       |
     +----------------------------------+
            |
            v
Existing repo tooling: pytest / Ruff / type checker / build system /
                     project scripts / local test services
            |
            v
Agent feedback (terse)   +   reviewer evidence (progressive disclosure)
            |
            v
         PR -> authoritative CI -> merge
```

| Responsibility | CI / Developer Infrastructure | Repository owners | Agent/plugin authors |
| --- | --- | --- | --- |
| Execution engine, status schema, cache semantics | Own | Integrate | Consume |
| Check catalog, eligible checks, lifecycle defaults | Define standards | Register/override within policy | Invoke |
| Runtime dependencies and test-specific setup | Provide preflight contract | Declare and maintain prerequisites | Follow permitted fixes |
| Behavior-to-test evidence | Provide schema, generation and validation | Supply intent, tests, optional annotations | Propose links/explanations |
| Agent interaction and custom workflows | Ship optional reference adapter | Extend as needed | Own |
| Required organizational gates | Retain in authoritative CI | Comply | Cannot bypass centrally |

The implementation should not require the agent to know whether a check uses pytest, Bazel, or a repository wrapper. The platform is responsible for normalizing invocation and results.

## 6. Core design

### 6.1 Check discovery and intelligent selection

Check discovery imports locally runnable checks from existing CI/build definitions where feasible, plus a lightweight repository manifest for gaps. Selection considers changed files, dependency graphs, check metadata, test history if available, and the lifecycle stage. Start with **deterministic path/dependency mappings**; evaluate learned or LLM-based ranking only if the pilot shows a need.

Every selected check records a reason (e.g., `services/auth/** changed` or `auth module depends on token parser`). Every deferred check records a reason (e.g., `trusted runner required`, `exceeds local budget`, `irrelevant to changed scope`). This is **selection parity with CI**, not execution parity: the system knows what exists, but does not run everything locally.

**Selection safety:** Conservatively include broader checks when dependency impact is uncertain; do not label selected-check success as full-regression success. Evaluate missed detectable failures against post-PR CI data.

### 6.2 Targeted execution

Run the smallest defensible set of check *units*, not just a shortened list of CI jobs: relevant test files/classes/targets where the test runner supports such selection. Reuse native build-system caches and incremental modes; parallelize independent work within laptop CPU/memory budgets. Track source state, command/configuration versions, duration, cache hits, and exit status.

V1 is **local-only**. Tests requiring unavailable architecture, trusted credentials, remote services, or impractical compute remain explicitly deferred to authoritative CI. A check can be locally eligible at pre-push even if it is too expensive after each edit.

### 6.3 Early preflight and execution reliability

Preflight has two levels:

- **Session readiness:** Low-cost, read-only inventory of runtime version, dependency installation, build tools, and known local services. Nonblocking; report gaps early to the agent/developer.
- **Check-specific readiness:** Validate prerequisites only for selected checks—service readiness, required configuration, local credentials, fixtures, disk and port availability. Cache stable results; re-probe volatile services.

Results must distinguish `passed`, `failed_code`, `blocked_environment`, `infra_error`, `flaky_or_inconclusive`, `skipped_policy`, and `stale`. A missing PostgreSQL server is not an assertion failure. Retry only when a failure class justifies it, with bounded attempts and backoff. Preflight diagnostics should avoid exposing credential values.

### 6.4 Direct feedback and remediation

Prefer deterministic evidence extraction (JUnit XML, test-runner JSON, compiler diagnostics, exit codes, known error signatures) and validated readiness probes. Return **one-line cause / evidence / next action / retry command**, with logs separately accessible.

An LLM can optionally suggest hypotheses for unknown failures or draft behavior explanations, but its suggestions are labeled **unverified** until grounded in a deterministic observation or human confirmation. The system must not silently run arbitrary shell commands, modify product code, weaken assertions, or disable tests to obtain a green result.

Illustrative feedback:

```json
{
  "check": "integration.auth",
  "status": "blocked_environment",
  "summary": "PostgreSQL is unreachable on the configured port.",
  "evidence": "Connection refused at localhost:5432; readiness probe failed",
  "next_action": "Start the repository's PostgreSQL service",
  "suggested_command": "docker compose up -d postgres",
  "requires_approval": true,
  "retry": "verify run --check integration.auth"
}
```

This does **not** assert PostgreSQL is stopped without confirmation; it reports the observation and suggests a bounded next step.

### 6.5 Behavioral verification artifact

The artifact should answer a reviewer's substantive question: **“Which intended outcomes did these new or existing tests actually exercise?”** It must not merely repeat counts of passing tests.

For each material expected behavior, V1 should produce an **evidence group** with: behavior/intent (from task or PR), relevant new and existing tests, concise explanation of the inputs/actions and assertions, execution outcome, source links, and any missing or ambiguous evidence. The mapping may be proposed using static/test inspection or an LLM, but **claims require source-linked assertion evidence**, and uncertain matches are marked `candidate` or `not established`.

**Illustrative reviewer summary (not proof of completeness):**

> **Token expiration:** Supported by `test_expired_token_returns_401` (new) and `test_token_expiry_boundary` (existing). Both passed; they exercise rejection of expired tokens and the expiration boundary with a fixed clock.  
> **Token refresh:** Supported by two refresh-endpoint tests (new); passed for valid and invalid credentials.  
> **Gap:** No identified test exercises concurrent refresh requests. Database integration checks were blocked locally and remain for CI.

Use progressive disclosure: a **single compact PR status / link** by default, with behavior groups visible on demand and raw provenance stored as a machine-readable artifact. Successful low-level checks (lint, type checks) need not be enumerated in reviewer prose. Avoid large PR bot comments and repeated updates. Report `evidence stale` if relevant files change after execution.

This is behavioral **evidence**, not formal verification or a guarantee that all specified behavior is covered. The exact source snapshot, test IDs/assertion locations, command/configuration, and run IDs must remain inspectable.

## 7. Lifecycle policy and extension model

The CI team publishes **recommended defaults** through pre-commit, pre-push, and agent-stop hooks, backed by one CLI. Hooks are entry points, not separate engines. Repo owners may tune eligibility and budgets within approved guardrails; developers may retain custom skills/plugins and invoke the CLI directly.

```yaml
# Illustrative policy — not an adopted standard
verification:
  session_start: { action: doctor, mode: advisory }
  agent_stop:    { profile: affected, mode: advisory, budget_seconds: 60 }
  pre_commit:    { profile: staged_fast, mode: blocking, budget_seconds: 15 }
  pre_push:      { profile: affected, mode: blocking, budget_seconds: 120 }
  pre_pr:        { profile: required_local, mode: report, budget_seconds: 300 }
```

`pre_commit` must verify the **staged tree** where feasible, not inadvertently certify unstaged edits. `pre_push` scopes to pushed commits, and a final pre-PR receipt binds to the actual workspace snapshot (tracked, staged, unstaged, and relevant untracked content). Local hooks are bypassable and **never** replace centralized merge enforcement.

A custom skill can call `verify plan`, `verify run`, and `verify report`. CI may ship a reference Claude Code plugin and installation path, but the platform's value does not depend on that plugin being the exclusive interaction surface.

## 8. V1 deliverables and boundaries

| V1 deliverable | Minimum acceptance condition |
| --- | --- |
| Versioned local `verify` CLI | Discover, plan, doctor, run, and report for pilot repositories |
| CI-derived check catalog | Registrations explain locally eligible versus deferred checks |
| Deterministic affected-check selection | Includes why each check is selected; conservative fallback for uncertainty |
| Targeted local runner | Structured status, safe timeouts/retries, cache reuse, bounded parallelism |
| Preflight & remediation feedback | Separates environment blockers from code failures and gives validated next actions |
| Behavioral evidence prototype | Groups new/existing tests by intended outcome with assertion/source links and uncertainty labels |
| Optional Claude Code adapter | Agent-completion hook and examples; no required migration from custom skills |
| Lightweight lifecycle defaults | Pre-commit and pre-push profiles with opt-in enforcement during pilot |
| Instrumentation | Measures usage, latency, pre-PR detections, accuracy, and downstream CI outcomes |

**Explicitly defer:** remote execution; sophisticated ML test selection; automated test generation; formal spec proving; full coverage across all languages; mandatory organization-wide installation; auto-remediation that requires privileges; automatic semantic claims without source evidence.

## 9. Adoption: design partners before general availability

Treat adoption as a **product risk and a formal launch gate**, not an afterthought. Select 2–3 backend teams with contrasting needs: frequent preventable PR failures, difficult local integration setup, and heavy agent usage with bespoke skills. A partner commits to real-task observation, feedback, and the option to contribute repository verification definitions.

### Pilot sequence

| Phase | What happens | Exit evidence |
| --- | --- | --- |
| A. Discovery & baseline | Observe agents/developers without the plugin; record commands, CI failures, setup pain, custom skills, elapsed time | Ranked pain points and baseline metrics |
| B. Co-development | Integrate thin CLI with 2–3 real repos; fix recurring diagnostic and selection gaps alongside teams | Repeated successful verification sessions on representative changes |
| C. Controlled evaluation | Compare similar tasks with and without the integration; inspect PR CI outcomes and behavior-report accuracy | Clear incremental benefit without unacceptable latency |
| D. Scale decision | Collect opt-in intent, maintenance ownership, and extension requests | Go / revise / stop decision supported by results |

Distribution can follow a paved-road model: easy installation, small demos based on actual failures, discoverable docs, and team-owned extensions. **Do not equate installed plugins with active value.** Share concrete before/after stories, not abstract check-selection features.

### Proposed go/no-go criteria (targets to calibrate with partners)

- **Adoption signal:** At least two of three partner teams voluntarily continue using the capability after the structured pilot.
- **Incremental value:** Demonstrated reduction in avoidable first-run PR CI failures or time spent on pre-PR verification/repair versus observed baseline.
- **Usability guardrail:** No material regression in median task-completion time from verification overhead; record P90 latency separately.
- **Trust guardrail:** Manual audit finds behavior-to-test explanations source-grounded, and the tool does not label blocked or stale evidence as verified.
- **Maintainability:** Repositories can add checks or remediation rules without CI engineers editing core code for every new case.

The numeric thresholds beyond the team-continuation signal should be set with partner baseline data, not invented in advance.

## 10. Measurement and evaluation

**Primary outcome:** Reduction in **avoidable post-PR CI rework per agent-generated change**, measured alongside **total time from coding start to PR-ready verification**.

| Measure | Definition / interpretation |
| --- | --- |
| Avoidable first-run CI failures | PR CI failures caused by checks that were locally eligible and could have detected the issue before PR |
| Time to actionable failure | Elapsed time from introduced defect (where observable) or verification request to useful diagnosis |
| Time to verified outcome | Time including setup, retries, agent fixes, and successful recheck—not just check runtime |
| Selection quality | Failures missed by chosen local checks but later detected by applicable CI checks |
| Diagnostic quality | Fraction of failure classifications/remediation suggestions confirmed correct in sampled runs |
| Evidence quality | Fraction of behavior-to-test claims supported by inspected assertions and execution proof |
| Activation and retention | Eligible agent sessions invoking verification; weekly voluntary repeat use |
| Cost guardrails | Added local CPU/memory, task wall time, retries, and excess test runs |

Use paired tasks, comparable repositories, and historical PR CI as reference evidence when feasible. Report changes in test workload and false negatives: improved PR CI success alone could come from running fewer or weaker checks. The pilot should explicitly compare **existing agent behavior without the plugin** against the platform workflow.

## 11. Risks, tradeoffs, and alternatives

| Risk | Mitigation |
| --- | --- |
| Agent already selects tests adequately | Baseline study and opt-in continuation gate; do not scale without differential value |
| Hooks slow or disrupt development | Stage-based budgets, fast defaults, advisory mode, transparent skips and timeouts |
| False confidence from narrow test selection | Selection reasons, deferred checks, stale flags, authoritative PR CI, false-negative monitoring |
| Misdiagnosis or destructive remediation | Deterministic probes, bounded retries, approval for mutations, hypothesis labels |
| Behavioral summaries hallucinate coverage | Link to actual assertions, run evidence, and source snapshot; label unsupported claims |
| Local environment drift / secrets exposure | Read-only preflight, environment constraints, sanitized logs, least-privilege execution |
| Plugin fragmentation / low adoption | Stable CLI and published contracts; optional integrations and decentralized extension |
| Maintenance grows per language/repo | Start with limited pilot repository types; configurable adapters and team ownership |

**Alternatives considered:** (A) Leave CI only at PR stage—minimal implementation, but preserves avoidable repair cycles. (B) Ship only a generic `verify` skill—low build cost but no shared selection, reliability, or evidence contract. (C) Mandate a monolithic Claude Code plugin—easy uniformity but high adoption friction. (D) Move all CI checks local—costly and often infeasible. (E) Build remote execution first—potentially useful later, but adds infrastructure cost before proving local-agent value. This RFC recommends a local-first shared capability with optional adapters.

## 12. Open questions and requested decision

**Open for design partners:**

1. What source of intended behavior is sufficiently reliable for V1: issue, PR description, agent plan, explicit developer annotation, or combinations?
2. How should behavior-to-test evidence be audited and corrected when mapping is uncertain?
3. Which check catalogs or CI definitions can be reused without bespoke integration per repository?
4. What is the right staged/uncommitted snapshot mechanism for reproducible local receipts?
5. How much lifecycle policy should be centrally enforced versus configurable, especially before commit and push?
6. What are acceptable execution budgets and check-selection false-negative rates for each pilot repository?
7. When, if ever, does a local verification gap justify a separate remote execution project?

**Decision requested now:** Approve a scoped discovery and design-partner pilot. Assign CI ownership of the engine and contracts, recruit 2–3 teams, build the minimum CLI and optional Claude Code adapter, and evaluate incremental benefit before mandating adoption or extending execution infrastructure.

## Appendix A. Illustrative CLI contract

```bash
verify doctor                        # low-cost session readiness
verify plan --base origin/main      # applicable checks and deferral reasons
verify run --profile fast           # frequent, targeted local checks
verify run --profile affected       # broader affected checks
verify run --check integration.auth  # one specific check
verify report --format json         # complete machine-readable evidence
verify report --format summary      # compact reviewer-focused summary
```

The JSON result should include a schema version, source snapshot and config identity, lifecycle stage, selected/deferred check IDs and rationales, normalized outcomes, diagnosis provenance, related test/assertion locations, observed behavior evidence, open gaps, and timestamps. The reporter must distinguish `executed and passed` from `selected but blocked`, `not selected`, and `outdated`.

## Appendix B. Relevant precedents (context, not direct proof of this design)

- **Spotify Engineering:** [Background Coding Agents: Predictable Results Through Strong Feedback Loops (Honk, Part 3)](https://engineering.atspotify.com/2025/12/feedback-loops-background-coding-agents-part-3) — verifier abstraction, concise parser output, agent stop hook.
- **Cursor:** [Create Verification Skill](https://github.com/cursor/plugins/blob/main/pstack/skills/create-verification-skill/SKILL.md) — launch/doctor/drive/evidence/cleanup and repo feature maps.
- **Google / academic:** [Regression Test Selection at Scale](https://research.google/pubs/regression-test-selection-at-scale-2/) — selective presubmit verification under constrained execution cost.
- **ISSTA 2026:** [To Run or Not to Run](https://arxiv.org/abs/2606.26978) — evaluates the cost-effectiveness of agent test execution; cautions against gratuitous reruns.
- **Google / ASE 2026:** [Catching Developers in the Flow](https://arxiv.org/abs/2610.07289) — agent-assisted remediation of presubmit CI failures; contrasts with this RFC's earlier loop.
- **Salesforce Engineering:** [How Salesforce Scaled Engineering AI Adoption Without Stifling Innovation](https://engineering.salesforce.com/how-salesforce-scaled-engineering-ai-adoption-without-stifling-innovation/) — shared Claude Code plugin with team-owned skills and adoption practices.

## Appendix C. Recommended automated hook workflow

The shared Claude Code plugin should automate verification as an idempotent state machine around the `verify` CLI. Hooks decide when the workflow advances, the `/innerloop:plan` skill supplies semantic intent, and the CLI owns check discovery, execution, snapshot identity, freshness, and pass/fail. A repeated hook invocation must reuse a matching receipt rather than execute the same checks again.

Claude Code does not expose native `PreCommit`, `PrePush`, or `PrePR` events. The plugin therefore combines Claude Code lifecycle hooks with optional native Git hooks. The Claude hooks cover operations initiated by the agent; native `pre-commit` and `pre-push` hooks cover actions from another terminal. Authoritative PR CI remains the final gate. See the [Claude Code hooks reference](https://code.claude.com/docs/en/hooks), [hooks guide](https://code.claude.com/docs/en/hooks-guide), and [Git hooks documentation](https://git-scm.com/docs/githooks).

### Automated lifecycle

```mermaid
flowchart TD
    A[Session starts or resumes] --> B[verify doctor]
    B --> C[User submits a code-changing task]
    C --> D[/innerloop:plan]
    D --> E[verify plan resolve and save]
    E --> F[Agent implements the change]
    F --> G[Mark prior evidence stale]
    G --> H[Stop or task completion]
    H --> I[Reconcile plan and run planned local checks]
    I -->|Actionable failure| J[Return evidence and next action to agent]
    J --> F
    I -->|Pass or explicit deferral| K[Create workspace receipt]
    K --> L[Commit attempt]
    L --> M[Verify exact staged snapshot]
    M --> N[Create commit receipt]
    N --> O[Push or create PR]
    O --> P{Receipt matches source and policy}
    P -->|Yes| Q[Reuse receipt]
    P -->|No| R[Run readiness checks]
    Q --> S[Authoritative PR CI]
    R --> S
```

### Plugin hook configuration

The following `hooks/hooks.json` is illustrative. The handler scripts contain the state and snapshot logic; the `if` expressions are fast filters, not the complete policy.

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/hooks/session-start.sh",
            "timeout": 15
          }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/hooks/prompt-context.sh",
            "timeout": 5
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Edit|Write|NotebookEdit",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/hooks/require-plan.sh"
          }
        ]
      },
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "if": "Bash(git commit *)",
            "command": "${CLAUDE_PLUGIN_ROOT}/hooks/pre-commit.sh",
            "timeout": 600
          },
          {
            "type": "command",
            "if": "Bash(git push *)",
            "command": "${CLAUDE_PLUGIN_ROOT}/hooks/readiness.sh",
            "timeout": 600
          },
          {
            "type": "command",
            "if": "Bash(gh pr create *)",
            "command": "${CLAUDE_PLUGIN_ROOT}/hooks/readiness.sh",
            "timeout": 600
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write|NotebookEdit",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/hooks/mark-dirty.sh"
          }
        ]
      }
    ],
    "TaskCompleted": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/hooks/completion-gate.sh",
            "timeout": 600
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/hooks/completion-gate.sh",
            "timeout": 600
          }
        ]
      }
    ],
    "PreCompact": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/hooks/checkpoint-state.sh"
          }
        ]
      }
    ]
  }
}
```

### Handler responsibilities

| Hook handler | Required behavior |
| --- | --- |
| `session-start.sh` | Run a fast, read-only `verify doctor`; discover configuration and prerequisites; load any active plan or receipt; add a concise status summary to Claude's context. It must not install dependencies, start services, or mutate the repository. |
| `prompt-context.sh` | Tell Claude to invoke `/innerloop:plan` when a prompt requests product-code changes and no current plan exists. The hook supplies context; it does not recursively start another Claude process. |
| `require-plan.sh` | Before the first `Edit`, `Write`, or `NotebookEdit`, block a code-changing task that has no active plan and direct Claude to invoke `/innerloop:plan`. Exempt explanatory, documentation-only, and other configured low-risk work. |
| `mark-dirty.sh` | Record that the workspace changed and any earlier evidence may be stale. Do not run tests after every edit. The completion and commit gates recompute the source snapshot so Bash or external edits are still detected. |
| `completion-gate.sh` | If an active plan exists and its source snapshot changed, run `verify plan reconcile`, execute the planned locally eligible checks, and create a workspace receipt. On an actionable failure, exit with code 2 and return concise evidence, an investigation target, and the smallest retry command. |
| `pre-commit.sh` | Run the complete locally eligible plan against the exact Git index snapshot. On success, create a receipt bound to the staged-tree hash, plan hash, verification configuration, and check results. |
| `readiness.sh` | For push and PR creation, accept a matching commit receipt immediately. If it is missing or stale, run only outstanding readiness checks and record explicit CI-only deferrals. |
| `checkpoint-state.sh` | Persist the active plan, last attempted snapshot, failure fingerprint, and receipt references before context compaction. |

### Workflow state and receipts

The engine should persist local, gitignored state such as `active-plan.json`, `workflow-state.json`, `last-attempt.json`, and snapshot-bound receipts. The minimum state progression is:

```text
NO_PLAN -> PLANNED -> DIRTY -> VERIFYING -> VERIFIED
                       ^           |
                       |           v
                       +-------- FIXING
```

A receipt is valid only when the source snapshot, plan, verification configuration, and required check set still match. The pre-commit receipt binds to the staged tree. The readiness trigger reuses that receipt when the resulting commit contains the same content; otherwise it executes the invalidated or previously deferred local checks.

### Completion and loop safeguards

`Stop` fires whenever Claude finishes a response, so `completion-gate.sh` must return immediately for question-answer turns, sessions without an active code-change plan, unchanged snapshots, already verified snapshots, and background work that is still running. It must inspect `stop_hook_active` and store the last attempted snapshot and failure fingerprint. If the same failure recurs without a source change, the hook should surface a blocker instead of continuing indefinitely.

`TaskCompleted` and `Stop` may both invoke the completion gate. Receipt lookup therefore precedes execution, making the second event a no-op. The same rule applies when a Claude `PreToolUse` hook and a native Git hook both protect the same commit or push.

Command hooks and the deterministic CLI should enforce production gates. Prompt- or agent-based hooks may assist ambiguous diagnosis during the pilot, but their output must not determine pass/fail or present a hypothesis as an observed cause.

---

*RFC note: This document is a proposed system design and evaluation plan, not a claim that the outcomes or pilot targets have already been achieved.*
