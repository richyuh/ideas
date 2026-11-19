# The Atlassian AI SDLC Index

- **Source:** [atlassian.com/blog/ai-sdlc-index](https://www.atlassian.com/blog/ai-sdlc-index)
- **Publisher:** Atlassian, with data from DX (getdx.com)
- **Read:** 2026-11-18 (page last updated 2026-11-18; living report, figures change)
- **Tags:** AI SDLC, developer experience, code review, metrics

## Summary

A recurring data report on how AI is changing the software development lifecycle, built from DX platform data on 340,000+ engineers at 500+ organizations. It tracks what developers use AI for, developer-experience sentiment from 2025 to 2026, time lost to friction, PR review load, AI-authored code share and survival, and bot review adoption. Sample sizes vary by section, and the time-loss figures are self-reported.

## Key takeaways

- **Friction costs about 20% of work hours** (~409 hours, ~51 days per year). CI wait and review wait lead at 86.5 hours each, then code comprehension (71.5), test wait (58), information-seeking (55), and dev environment toil (47.5).
- **More AI code means slower reviews.** Median time to first human review rises from 31.2 minutes at 20–40% AI-authored code to 97.8 minutes at 80%+. Reviews per person per month went from 12.8 (Jan 2026) to 22.3 (Oct 2026).
- **AI code volume is up, survival is mixed.** AI-authored share rose from 49% (Apr 2026) to 69.7% (Oct 2026). The share surviving into commits fell from 79.9% to 58.9% (Sep) before recovering to 68.2% (Oct).
- **Sentiment is mixed.** Documentation improved most (47.7% to 60.4% favorable). Change confidence, local iteration speed, incident handling, and clear direction declined.
- **Bug fixing is the top AI use** at 20.2% of sessions, ahead of architecture and planning, code review, documentation, and refactors (each ~10–12%).
- **Innovation time is creeping up**: ~58% of time on new features (Sep 2026) vs. 55.3% (Jan 2025). Bot review coverage grew from 15.3% of PRs (Nov 2025) to a 21.6% peak (Aug–Sep 2026).

## Notes

- Review is the bottleneck AI makes worse: review wait is already a top friction source, and AI-heavy PRs wait three times longer for a first review. This supports the case in [Agent-Orchestrated Software Verification](../agent-orchestrated-software-verification.md) for moving inspection off human reviewers.
