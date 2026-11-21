# RFC: Specbase — A Compiler for Verifiable Software Engineering Intent

**Status:** Proposal

**Date:** 2026-11-20

**Proposed artifact:** `specbase.md` in the ideas repository

> **From codebase to specbase:** engineers maintain a versioned definition of what a system must do and guarantee; a compiler coordinates the production, verification, and evolution of its implementation.

## Problem

Software development has become much faster at generating code, but the surrounding workflow remains manual and imperative. An engineer writes a specification, turns it into tickets, prompts agents, supervises execution, checks tests, resolves misunderstandings, and manually updates plans when requirements change. The spec and implementation drift apart. Tickets describe work completed, not guarantees established.

Today's spec-driven tools improve document organization and task generation, but a chain of generated Markdown, tasks, code, and tests does not independently establish that the delivered system matches the original intent. A coding agent can misunderstand the requirement, implement that misunderstanding, and generate tests that faithfully reinforce it.

The missing abstraction is not a better prompt. It is a **specification compiler** that treats intent as an authoritative, versioned artifact and all downstream plans, PRs, and verification evidence as derived, traceable artifacts.

## Vision

An engineer describes the desired system declaratively: capabilities, behavioral contracts, non-negotiable invariants, constraints, operating assumptions, acceptance criteria, and tradeoffs. The system formalizes that intent, asks for decisions where meaning is unresolved, and locks the approved contract. It then compiles that contract into a milestone/feature dependency graph, estimates complexity and uncertainty, dispatches implementation agents, and independently verifies the results. A pivot becomes a new spec revision, followed by incremental recompilation and re-verification of affected artifacts.

The desired interaction changes from:

> Create tickets; prompt an agent; inspect its PR; run checks; correct drift; update the plan; repeat.

To:

> Here is Specbase revision N. Reconcile the implementation to this approved contract, return evidence, and escalate only decisions outside your authority.

**Core invariant of the platform:** an implementation agent cannot silently redefine a requirement, weaken a verification obligation, or certify its own unverified work as compliant.

## Goals and non-goals

**Goals**

1. Make engineering intent a first-class, version-controlled, reviewable system artifact.
2. Separate human-approved requirements from generated designs, task plans, code, and tests.
3. Bind each requirement to explicit assumptions and verification obligations.
4. Compile specs into a dependency-aware milestone/feature graph, then into agent-executable work and PR artifacts.
5. Require independently collected, scoped evidence before a requirement is marked satisfied.
6. Detect, explain, and reconcile implementation drift and changes in upstream requirements.
7. Reduce human orchestration and review load without increasing escaped defects or weakening safety controls.

**Non-goals**

- Claiming that arbitrary English requirements can be automatically understood without ambiguity.
- Proving all properties of arbitrary applications or guaranteeing that any generated code is universally correct.
- Replacing every existing test, CI system, reviewer, or production observability practice.
- Making every architectural choice deterministic; multiple implementations can satisfy the same contract.
- Treating a passing test suite, an agent's self-assessment, or structural traceability as mathematical proof.

## Artifact model

| Artifact | Authority and role | Can agents alter it unilaterally? |
| --- | --- | --- |
| **Intent spec** | Human-readable system goals, semantics, risks, non-goals, and policy decisions | May propose edits; approval required for authoritative changes |
| **Contract IR** | Typed representation of behaviors, invariants, constraints, assumptions, and acceptance obligations | May generate candidates; cannot approve their meaning |
| **`spec.lock`** | Approved, versioned contract snapshot with schema, requirement IDs, hashes, provenance, verification policies, and tool versions | No silent changes; controlled revision only |
| **Milestone/feature DAG** | Derived dependency graph, feature contracts, complexity estimates, acceptance gates, and traceability | Yes, subject to parent-contract validation |
| **Agent work / PRs** | Replaceable implementations, migrations, generated tests, and supporting artifacts | Yes, within granted permissions |
| **Assurance records** | Independent results tied to contract revision, commit, verifier, assumptions, and environment | Produced by trusted verification systems, not editable by the implementer |

The **specbase** is the authoritative set of approved intent and contracts plus the provenance graph connecting them to derived work and evidence. The **codebase** remains executable reality; authoritative intent does not make a broken deployment correct.

## Compilation pipeline

```text
Natural-language intent + external requirements
                 |
       Intent formalization
       (LLM proposes; validator challenges)
                 |
       Human semantic approval
                 |
     Typed contract IR + spec.lock
                 |
     Refinement / planning engine
                 |
       Milestone / feature DAG
       (dependencies, complexity,
        proof obligations, gates)
                 |
      Agent execution scheduler
                 |
       Code / tests / PRs
                 |
  Independent verification + evidence
                 |
  Reconciliation to approved spec revision
                 |
   Verified delivery / escalation
```

### 1. Capture and formalize intent

Each requirement should identify at least: stable ID, owner/authority, rationale, scope, observable behavior, forbidden behavior, assumptions, external dependencies, acceptable exceptions, importance, and acceptance criteria. Requirements can reference existing APIs, consumer contracts, regulations, production incidents, and policy decisions as independent sources.

The formalization front end proposes a typed contract, but *must not silently resolve consequential ambiguity*. For example, "never charge twice" requires a shared definition of a logical payment, committed side effect, retry, refund, and external payment-provider behavior. The system presents representative examples, adversarial counterexamples, missing assumptions, and conflicts for explicit semantic approval.

A compiler can prove properties of a formal model. It cannot generally prove that the model faithfully captures what a person meant. The intent-to-contract boundary is a trust and governance problem, not merely a parsing problem.

### 2. Validate and lock the contract

The compiler checks supported contracts for type validity, contradictory constraints, unresolved references, explicit environmental assumptions, and whether critical obligations have enforcement mechanisms. It reports **approved**, **needs decision**, or **inconsistent** rather than inventing missing semantics.

A `spec.lock` snapshot binds approved requirement IDs and canonical contract hashes to their origin, approvers, analysis scope, verification obligations, relevant policy versions, and tool versions. The lockfile is *not* a substitute for human-readable intent, nor a guarantee that every important requirement was captured. It prevents approved obligations from being silently reinterpreted downstream.

### 3. Compile into milestone and feature obligations

The planner generates a DAG of deliverable capabilities, their dependencies, release order, acceptance gates, and bounded work units. Every child feature inherits applicable parent invariants and must explain how its outputs contribute to satisfying the parent contract. This is a **refinement** relationship, not just a link from a ticket to a paragraph.

For a parent guarantee `G`, feature guarantees `F1..Fn`, and stated environmental assumptions `A`, a strong compositional obligation is:

`A AND F1 AND ... AND Fn => G`

Where the supported formal language allows it, the compiler can prove this implication; elsewhere it records weaker evidence and unresolved proof obligations. Passing every child feature's tests does **not** alone establish the parent invariant, especially for cross-feature interactions.

The compiler also checks *coverage* of declared requirements: no requirement can disappear from the plan without an explicit disposition. This does not establish completeness of the original human intent.

### 4. Assess complexity before implementation

Every requirement and feature has an evidence-backed **complexity profile**, not an unexplained scalar:

- **Technical:** state space, concurrency, algorithms, precision, domain novelty.
- **Integration:** service boundaries, consumers, dependencies, legacy interfaces.
- **Verification:** oracle availability, ability to model properties, observability, test cost.
- **Operational:** rollout, migration, blast radius, rollback, production coupling.
- **Uncertainty:** missing information, unproven assumptions, unfamiliar components.

Report 1–5 ordinal scores with rationale, evidence, confidence, and unknowns. Keep **importance, risk, effort, and uncertainty separate**; they are not interchangeable. High uncertainty can trigger a feasibility spike. High verification complexity can trigger stronger design review or instrumentation. High blast radius can trigger staged rollout. Decomposition reduces work-unit size but does not magically reduce total project complexity.

Scores are calibrated against actual agent retries, rework, unexpected dependencies, verification costs, deployment incidents, and human interventions. The compiler must eventually learn when its estimates were wrong.

### 5. Generate and execute implementation work

Agents choose implementation strategies within the approved constraints. The scheduler creates tasks and PRs from the DAG, respects dependencies and permitted parallelism, tracks cost/budgets, retries known recoverable failures, and escalates blocked decisions. Plans and prompts are transient compiler outputs rather than manually maintained authorities.

A PR must identify the exact spec revision, requirements it claims to satisfy, affected dependencies, generated code and tests, verification obligations, and evidence links. A PR cannot close a requirement by simply asserting completion.

### 6. Verify independently

Each important requirement is associated with a **contract**, **verification method**, and **evidence record**. The method depends on the property:

| Property | Possible checks | Limits |
| --- | --- | --- |
| API behavior and compatibility | Contract tests, schema checking, consumer-driven contracts | Samples and modeled consumers may be incomplete |
| Safety invariants and state transitions | Model checking, SMT, proof assistants, property-based tests | Formal proof covers the model and assumptions; tests cover executions |
| Security/policy boundaries | Policy engines, static analysis, sandbox and permission controls | Coverage depends on the threat model and trusted boundary |
| Performance/cost budgets | Controlled benchmarks, load tests, runtime SLO monitoring | Workload and environment can differ from production |
| Cross-service behavior | Integration tests, fault injection, traces, staged rollout | No finite suite proves all distributed executions |

Evidence statuses must be **PROVED** (with stated formal scope and assumptions), **TESTED** (specified empirical checks passed), **REFUTED** (counterexample or failed check), or **UNVERIFIED** (missing/inconclusive evidence). A tested property is never silently promoted to proved; an unverified blocking obligation cannot be certified as satisfied.

The verification kernel should be kept as small and auditable as possible. Coding agents may generate candidate tests, models, or proof scripts, but their own claims are not the acceptance authority. Verification definitions that the agent is allowed to edit cannot serve as the sole independent gate for that agent's work. Where appropriate, use trusted policy/check definitions, external consumer contracts, independently authored examples, and fixed benchmark suites.

### 7. Reconcile and incrementally recompile

A pivot changes the spec, not just a ticket or prompt. A new approved revision triggers semantic diffing:

1. Identify changed contracts, assumptions, and verification obligations.
2. Invalidate affected feature plans and assurance records using dependency/provenance edges.
3. Recalculate complexity and reorder impacted milestones.
4. Regenerate or repair only affected tasks and implementations.
5. Reverify impacted obligations before claiming conformance to the new revision.

For example, changing a p95 verification-latency budget from 60 to 30 seconds affects performance benchmarks, scheduling choices, potentially execution architecture, and corresponding PRs. It need not invalidate unrelated API guarantees if the dependency model establishes their independence. Unknown impact must fall back to broader verification, not optimistic reuse of stale evidence.

Distinguish **implementation drift** (code no longer satisfies an approved contract, which can be repaired autonomously) from **intent change** (the approved contract itself changes, which needs authorized revision).

## Worked example: agentic pre-PR verification

**Written intent:** "No change should be presented as fully verified unless every applicable mandatory check has passed. Unknown check dependencies must not cause silent omission."

**Approved contracts (illustrative):**

- `INV-001`: For a given change and approved mandatory-check policy, reporting `FULLY_VERIFIED` implies all checks classified as mandatory have recorded passing results for the corresponding revision.
- `INV-002`: If the dependency mapping is incomplete, the system must fail closed or run the policy-defined conservative fallback; it may not silently certify selected checks as exhaustive.
- `PERF-001`: For eligible workloads, aim for a measured p95 feedback latency at or below a separately approved threshold.

**Possible feature DAG:** check discovery → policy-based selection → execution/result collection → status certification; with dependency-uncertainty fallback and independent selector-validation as cross-cutting obligations.

**Critical semantic caveat:** `INV-001` can be satisfied vacuously if the approved policy defines no mandatory checks. The specbase must independently validate the check policy's authority and intended coverage. No compiler can infer an unknown organization-wide definition of "required" from code alone.

**Candidate assurance:** schema validation + rule-engine tests + randomized graph scenarios + policy-protected CI + a sampled full-suite comparison + deployment monitoring. This supports a scoped claim; it is not a universal correctness proof.

## Control plane and trust boundaries

Use **probabilistic agents for search, candidate formalization, architecture, decomposition, implementation, and debugging**. Use **deterministic mechanisms for schema checks, immutable identifiers, state transitions, policy enforcement, dependency invalidation, evidence attestation, and declared acceptance gates**. Formal solvers or proof checkers can establish stronger guarantees for supported properties; their soundness and modeling assumptions become part of the trusted boundary.

An agent must not be able to edit an invariant, its only verifier, and the evidence status in one unchecked transaction. Critical policy exceptions, scope changes, weakened obligations, production permissions, and high-risk releases require explicit authorization. Sign/hash evidence against exact source revision, code commit, verifier identity/version, relevant policy, and execution environment.

The system may recommend a design; it does not have authority to redefine success to match the generated implementation. Trust must be earned through independent evidence and narrow privileges, not through more confident agent prose.

## MVP and staged validation

Start with **one repository and a constrained contract vocabulary**, not a universal English-to-code compiler. A strong initial domain is a small durable-job-processing service with idempotency, retry, and transition guarantees, or a narrowly bounded part of the pre-PR verification system.

1. **M0 — Gold-standard cases:** Curate 10–20 requirements with human-approved semantics, examples, known counterexamples, and existing implementations/faults. Establish a manual answer key.
2. **M1 — Contract front end:** Capture intent, generate typed contract candidates, surface ambiguities, require approval, and produce canonical versioned lockfile snapshots.
3. **M2 — Refinement graph:** Generate feature DAGs with obligations, provenance, complexity/uncertainty profiles, and automatic coverage checks. Demonstrate an intentionally missing invariant is detected or flagged as unknown.
4. **M3 — Agent compiler and gates:** Produce PRs in a sandbox branch, run externally governed checks, and emit scoped assurance records. Inject faulty implementations and test whether blocking requirements reject them.
5. **M4 — Incremental spec revisions:** Change approved constraints, invalidate only affected evidence, regenerate work, and demonstrate no silent contract weakening.
6. **M5 — Controlled comparison:** Compare against conventional spec-to-ticket/agent workflows on equivalent tasks, including escaped faults and human intervention costs.

Ship the prototype only if it provides measurable assurance beyond generated tickets/tests. Do not invest in autonomous orchestration before the verifier and change-propagation model are credible.

## Success metrics

Primary metric: **human interventions per independently verified requirement delivered**, interpreted alongside defect and coverage measures so removing review does not simply hide failures.

Guardrail and supporting metrics:

- Escaped defects and critical policy violations per delivered change.
- False certification rate on seeded faults and historical incident reproductions.
- Fraction of critical requirements with explicit contracts and independent verification obligations.
- Proof-vs-test-vs-unverified distribution, with the verified scope recorded.
- Spec-to-implementation drift detected and time to repair.
- Percentage of pivots executed as approved spec diffs rather than ad hoc task edits.
- Precision of dependency invalidation and cost of unnecessary re-verification.
- Engineering hours, agent cost, rework cycles, lead time, and human review minutes per accepted capability.
- Calibration of complexity scores and forecasted implementation/verification effort.

Do not optimize for generated PR count or raw requirement count. A reduction in human review is a success only when assurance quality is maintained or improved.

## Risks and tradeoffs

- **Formalization can encode the wrong meaning.** Mitigate with human semantic approval, negative examples, counterexample review, and independent domain sources.
- **A formal contract can be incomplete or vacuously satisfied.** Check satisfiability/non-vacuity, test adversarial omissions, and surface uncovered assumptions; never claim absolute completeness.
- **Correlated verifier failures.** Separate implementation authority from acceptance policy; use independently sourced contracts, mutation testing, and adversarial evaluations.
- **Formal methods are expensive or inapplicable to some requirements.** Support graduated assurance instead of pretending every property can be proved.
- **Dependency graphs can be wrong.** Record provenance, use conservative invalidation, and periodically compare incremental checks with full validation.
- **Specification maintenance may become another form of coding.** Keep the authoritative layer compact and focused on guarantees; generate implementation details and operational plans downstream.
- **Spec lockfiles can become stale.** Couple revisions to CI/deployment attestation and continuous production reconciliation.
- **Organizational adoption is nontrivial.** Begin as an overlay on current Git, CI, review, and incident workflows rather than requiring a wholesale migration.

## Alternatives and existing foundations

- **Spec-driven development** (e.g., [GitHub Spec Kit](https://github.com/github/spec-kit), [OpenSpec](https://github.com/Fission-AI/OpenSpec)) organizes plans and tasks; Specbase additionally proposes independently enforced contract refinement, provenance, and evidence-based acceptance.
- **Traditional CI and contract testing** verify specific behaviors; Specbase binds those checks to approved, versioned system-level obligations rather than leaving them as an unstructured set of jobs.
- **Formal methods** ([TLA+](https://lamport.azurewebsites.net/tla/tla.html), [Alloy](https://alloytools.org/), SMT/property testing) address supported properties; they do not automatically validate human intent or real-world assumptions.
- **Translation validation and verified compilation** ([Alive2](https://github.com/AliveToolkit/alive2), [CompCert](https://compcert.org/)) are useful inspirations for validating each refinement rather than trusting a probabilistic generator.
- **Existing project notes:** [Agent-Orchestrated Software Verification](./agent-orchestrated-software-verification.md) can provide the assurance backend; [Loop Engineering](./loop-engineering.md) can provide autonomous execution and escalation. Specbase supplies the authoritative contract and refinement model above them.

## Open questions

1. What minimal contract IR is expressive enough for real backend systems but narrow enough for sound, practical checking?
2. Which guarantees can be proved compositionally across generated feature boundaries, and how is the refinement mapping represented?
3. How can we detect missing requirements, vacuous guarantees, or incorrect environmental assumptions before implementation?
4. Which verifier definitions must be independently authored or protected, versus safely generated by agents?
5. How do we maintain semantic IDs and incremental invalidation when requirements are split, merged, or reworded?
6. When is a tested obligation sufficient, and who sets the required assurance level for a given risk?
7. How should the compiler handle design freedom, competing architectures, and cost/performance tradeoffs without embedding arbitrary choices in the spec?
8. What is the minimum amount of human semantic review that yields reliable outcomes without making specification authoring the new bottleneck?
9. How do we measure correctness and economic value against a strong baseline of modern agentic coding with conventional CI?

## Proposed decision

**Proceed with a research prototype of the contract front end, refinement graph, and independent assurance gate; defer general-purpose autonomous software-factory orchestration.**

The thesis is falsifiable: if bounded requirements cannot be formalized into useful contracts, if refinement cannot preserve meaningful obligations, or if independent verification costs more than the review it removes without reducing risk, the concept should pivot or stop. If it works, expand the supported contract vocabulary and domains incrementally.

> The codebase is an executable implementation. **The specbase is the maintained engineering intent, and the compiler must earn the right to declare the two aligned.**
