# A Look Inside the Factory

- **Source:** [thebuildingmachine.com/p/a-look-inside-the-factory](https://thebuildingmachine.com/p/a-look-inside-the-factory)
- **Publisher:** The Building Machine (Calvin Grunewald)
- **Read:** 2026-11-19 (published 2026-11-19)
- **Tags:** software factory, coding agents, verification, code review, autonomy

## Summary

A first-person account of building a "software factory" on top of a custom coding agent harness (Astrodev, later archdev). An overseer plans work as a graph of Tasks, workers write code in their own worktrees, and a local daemon runs lint, tests, policy checks, and adversarial review on each pushed commit before anything reaches GitHub. The author covers the design decisions, how they changed, and why the factory is now moving to the cloud. The evidence is one author's experience and internal metrics from one company's monorepo; none of it is independently verified.

## Key takeaways

- **The harness was built by an agent from a spec.** Codex turned a ranked list of about 30 commonly used Claude Code features into 188 tasks, and a roughly two-day unattended run checked off 130 of them. Each task had to pass type, lint, and test gates before commit.
- **Real end-to-end tests made that autonomy possible.** The author calls PTY tests that drive the actual TUI "the key trick."
- **The Task is the unit of work.** Tasks can be verified independently and automatically, and one PR per Task is easier to review and audit than stacked PRs.
- **Nothing reaches GitHub unverified.** The daemon runs the repo's lint, tests, policy checks, and multi-agent adversarial review on the exact commit the worker pushed, with up to two more fix-and-review rounds. After push it watches the PR and fixes review comments, CI failures, and merge conflicts.
- **Cheaper review-and-fix configurations cost quality.** In an internal eval of six configurations on 19 cases, the current one fixed all 9 high and 14 medium findings with no regressions. A configuration about 3.5x cheaper per fixed finding deleted a public wrapper it should have left alone.
- **Autonomy is a setting, and the default keeps a human in the loop.** `factory.autonomy` has `confirm` (default) and `auto` modes. Before it existed, workers sent the overseer 151 decision requests in about 21 hours. Humans still own credentials, spending, live infrastructure (including merges), irreversible actions, and product decisions.
- **Throughput is high and compute is the limit.** The factory has merged 1,940 PRs into the monorepo since August 21, peaking at 251 in one week. Two machines running workers and daemon jobs are still not enough.

## Notes

- This is a working instance of the pipeline argued for in [Agent-Orchestrated Software Verification](../agent-orchestrated-software-verification.md): inspection runs before a human sees the PR, against the exact commit, and the fixing agent is checked by a separate adversarial reviewer.
- The eval result is the part to remember when tuning cost: the cheap configuration failed by acting on a finding it should have declined, which a count of findings fixed does not show.
- The author warns that one-shotting a complex system "piles up design debt that you pay down later," and that agent plans can carry ambiguity or mistakes the human did not catch.
