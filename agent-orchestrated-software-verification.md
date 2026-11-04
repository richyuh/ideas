# Agent-Orchestrated Software Verification

## Problem

Modern CI systems primarily execute predefined jobs: builds, tests, linters, security scans, and performance checks. A green pipeline confirms that the configured jobs passed, but it does not necessarily establish:

- Which product claims were verified
- Whether the correct checks were selected
- Whether tests adequately demonstrate the change
- Which downstream systems may be affected
- What risks remain outside automated coverage
- How much confidence a reviewer should place in the results

As coding agents increase the volume and speed of implementation, human reviewers cannot remain the primary inspectors of every change. Software delivery needs a verification system that converts available context, automated measurements, and specialized inspection into a concise acceptance case.

## Vision

The verification system is **agent-orchestrated, policy-governed, and evidence-driven**.

A verification agent coordinates the process. It interprets the change, identifies affected systems, selects appropriate checks, invokes specialized review agents, and consolidates the results. Deterministic tools perform measurable checks. Organizational policies enforce mandatory controls. Humans adjudicate material uncertainty and residual risk.

> **Specialized agents perform inspection. Verification tools establish measurable conformance. Humans adjudicate the resulting evidence according to risk.**

## How it works

### 1. Establish the provisional contract

The system gathers the best available context:

- PR description
- Feature requirements
- System specifications
- API and schema contracts
- Architectural standards
- Performance budgets
- Security and operational policies
- Existing tests and production history

When no authoritative specification exists, the PR description becomes a provisional contract. The system records the source and authority of every claim rather than treating inferred assumptions as fact.

### 2. Analyze impact and risk

The verification agent examines:

- Changed components
- Interfaces and dependencies
- Known consumers
- Data and schema modifications
- Security boundaries
- Critical performance paths
- Deployment and rollback complexity

It then classifies the change and proposes a verification plan. A documentation update should not receive the same scrutiny as an authorization change or database migration.

### 3. Execute the verification plan

The plan combines existing checks with change-specific investigation:

- Functional and acceptance tests
- Regression and compatibility checks
- Performance benchmarks
- Architecture policies
- Security and privacy analysis
- Migration and rollback validation
- Failure-injection scenarios
- Generated edge cases and targeted probes

Fast checks should be available locally during implementation. Authoritative checks run in controlled CI environments before merge. Production smoke tests and canary analysis continue verification after deployment.

### 4. Invoke specialized review agents

Specialized agents examine properties that deterministic checks cannot fully measure:

- Test adequacy
- Architectural compatibility
- Downstream impact
- Maintainability and coupling
- Security and privacy
- Performance and cost
- Operability and rollout risk

Their findings must include severity, confidence, supporting evidence, the relevant requirement or standard, and whether the issue should block the change.

### 5. Produce an evidence package

The system consolidates all results into a decision-oriented report:

```text
Risk: Medium

Change claims
✓ 5 of 5 described behaviors demonstrated

Regression surface
✓ Existing test suite
✓ Public API compatibility
✓ Known consumers
✓ Performance baseline

Specialized inspection
✓ Security: no material findings
△ Architecture: one medium-confidence concern

Context limitations
– PR description used as provisional contract
– Dependency graph may be incomplete

Recommendation
Human review required
```

The conclusion must reflect the limits of the evidence. When context is incomplete, the system should say:

> **No regressions were detected within the verified surface.**

It should not claim universal correctness.

## Decision model

The system supports three primary outcomes:

- **Approve:** Required verification passed and residual risk is acceptable.
- **Needs work:** A specific defect or evidence gap must be resolved.
- **Needs deeper review:** Material uncertainty requires direct human or specialist inspection.

Low-risk, well-understood changes may be eligible for automatic merge. High-risk, novel, or poorly specified changes require human adjudication.

## Governance

The verification agent may select and generate checks, but it does not define what the organization is willing to accept. It cannot:

- Remove mandatory checks
- Weaken acceptance criteria
- Change performance thresholds
- Approve policy exceptions
- Silently reclassify risk
- Treat inferred assumptions as authoritative
- Approve an implementation it produced

Humans and system owners define contracts and risk tolerances. Policy enforces minimum controls. The verification agent determines how to investigate the change within those boundaries.

## Trust and calibration

The verification system must itself be measured. The organization should track:

- Known defects detected
- Escaped production failures
- Review-agent false positives
- Flaky checks
- Benchmark stability
- Mutation-test results
- Policy bypasses
- Human overrides of agent recommendations

Production incidents should improve requirements, tests, review agents, and risk policies.

## Outcome

The objective is not to prove that software is universally correct. It is to make merge confidence explicit and defensible:

> **For this change, these claims were evaluated, these measurements were performed, these risks were inspected, and these uncertainties remain.**

This shifts the human reviewer from exhaustive code inspector to accountable evidence adjudicator—while preserving human ownership of the final risk decision.
