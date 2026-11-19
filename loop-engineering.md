# Loop Engineering: Harness Design for Agent Autonomy

## Problem

Coding agents are capable enough to finish most tasks, yet the human still does much of the work around them:

- **Premature stops.** The agent halts on an open question, a permission block, or a tool error, and waits for a human.
- **Prompt crafting.** Delegated work only goes well when someone writes a careful, well-defined brief.
- **Reading output.** Results arrive as long prose in one confident tone, and the human reads all of it to find the two lines that matter.
- **Relaying.** The human copies CI failures to the agent, one agent's output to another, and the agent's requested commands to a terminal, adding no judgment along the way.

These are harness failures, not model failures. Stopping costs the agent nothing, so it stops. Nothing forces a brief or a report into a useful shape. Nothing connects the agent to the things the human keeps carrying to it.

## Vision

The human stops writing prompts and writes loops. The loop generates the prompt, does the work, checks it, and retries.

> **Code owns control flow. The model owns content. The human owns decisions.**

## How it works

A loop has six parts. Each one removes a source of friction.

1. **Trigger.** A rough ask, a schedule, or a failing check starts the loop.
2. **Brief.** A higher tier model turns the ask into a well-defined prompt. It fills gaps by investigating the repository first, then by choosing a default and recording it as an assumption, and asks the human only for decisions that are theirs to make. The harness enforces a fixed shape and rejects a brief with an empty field: goal and why, files to read, what is ruled out, constraints, done criteria a check can verify, what to do when blocked, what to return.
3. **Work.** A worker agent carries out the brief.
4. **Check.** Code decides pass or fail. The worker's opinion that it is finished does not count.
5. **Feedback.** Failure output becomes the input to the next iteration, with no human in between.
6. **Exit or escalate.** The loop ends when the check passes, when the budget is spent, or when it reaches a decision only the human can make.

### Preflight

No implementation starts until the loop is shown to be viable:

- The check runs now, and it fails now. A check that is already green proves nothing.
- The agent dry-runs every tool and permission it will need.
- Open questions are asked in one batch at the start.
- The budget and the escalation path are set.
- One trivial change travels the whole loop end to end.

When preflight fails, the agent repairs the loop (writes the missing test, requests the access) and tries again before it asks for help. Preflight is timeboxed. Cheap probes beat long plans, because agents plan confidently about things they have not touched.

### When the agent is blocked

A fixed table maps each kind of block to a next action:

- **Open question:** take the default and record it, or park the question as its own task and continue the work that is not blocked.
- **Tool error:** diagnose, retry, and fall back before giving up.
- **Permission block:** continue if the action is on the pre-granted list. Otherwise send a one-line approval request and keep working on the rest.

### Output

The design assumes the human's reading budget is near zero.

- Success is silent. Only escalations, assumptions, and out-of-brief changes reach the human.
- Status is state: task, pass or fail, link.
- A request to the human is one question with two to four options.
- The agent shows the thing itself (a running app, a screenshot, a diff) instead of describing it.
- Reports have a hard length cap, and overflow goes to a file.
- Verified results are marked apart from claims. Failures pass through unsummarized, with links to raw evidence.

### No relays

If the human adds no information or judgment when passing something along, that hop should not exist. The agent gets direct access to whatever the human was carrying. Every "please run this for me" is a missing tool.

## Enforcement

Prompt files such as CLAUDE.md, skills, and memory notes are advisory. The agent can ignore them, misread them, or lose them as its context fills. A constraint that must hold needs a mechanism the agent cannot talk its way past. From weakest to strongest:

1. **Prompt files and skills.** A request to the model.
2. **Output schemas.** A malformed brief or report is rejected.
3. **Hooks.** Code runs before and after tool calls and when the agent tries to stop, and it can block.
4. **Permission rules and tool wrappers.** The wrong action is not on offer.
5. **Sandbox and scoped credentials.** The wrong action is impossible.
6. **Gates outside the agent.** Pre-push checks, CI, and branch protection, which the agent cannot bypass.

Anything that must hold goes at level 3 or above. Prompt files are for guidance where some deviation is tolerable. When a rule in a prompt file is broken twice, it moves up the ladder.

## Learning loop

The model does not learn between runs, so the harness must. Every run produces two outputs: the result, and an improvement to the loop. The inner loop gets the task done. The outer loop makes the inner loop better.

1. **Capture.** Every run leaves a structured trace, on success or failure: escalations, retries, tool errors, assumptions taken, time and cost per stage, and the human's answers.
2. **Attribute.** A separate reviewer agent decides which part of the loop was at fault: the brief, a tool, the check, the exit rule, or missing knowledge. Every friction is treated as a harness bug.
3. **Encode.** The lesson is stored as high on the enforcement ladder as it can go. If it can be code, it is not stored as text.
4. **Validate.** The change is replayed against past tasks. It stays if the numbers improve and is reverted if they do not. Attribution from one run is guesswork, so a pattern appears twice, or survives replay, before it becomes a rule.
5. **Retrieve.** The next task is matched to its type and starts from the relevant recipes and rules. A lesson that is not loaded at the right moment was never learned.
6. **Prune.** Stale and unused lessons expire, and a recipe is checked against the current code before reuse.

Three common failures show how a lesson gets encoded:

- **Wrong direction.** The brief recipe gains the missing constraint, and an early checkpoint makes a wrong approach fail in minutes.
- **Tool called the wrong way.** The tool is fixed, with a clearer error message or a wrapper that makes the wrong call impossible.
- **Inefficient path.** After a success, the reviewer extracts the shortest path in hindsight and saves it as a recipe or a single tool.

This needs task types, so that lessons transfer, and a set of past tasks to replay.

The principle is to never pay for the same lesson twice.

## Governance

Some hops are deliberate walls. Credentials, spending, production changes, and irreversible actions stay with the human, reduced to a one-click approval. The loop may not weaken its own check or change its own exit condition to get to green. Changes the outer loop makes to the harness pass their own checks, and changes to policy or permissions need human approval.

Determinism has a cost. Every hard-coded rule is a judgment the agent can no longer adapt. Hard-code the stop conditions and the verification. Leave the method of doing the work to the model.

A loop is only as good as its check. Where no mechanical check exists, as with design taste or the choice of what to build, a loop either spins or converges on the wrong thing. Whatever the human does not read, only the checks stand behind.

## Measuring it

- Relay hops per task
- Escalations per task, and the share that were truly the human's to decide
- Words the human reads per merged change
- Preflight failures, by cause
- Defects that passed the check
- Cost, time, and escalations for the Nth run of a task type, which should fall

Whatever the human had to read or relay by hand is a list of missing checks and missing tools.

### Signals in pull request data

Each pull request is a record of one loop run. The places where a human stepped in, or where work was redone, point at the part of the loop to fix.

| Signal | What it points to |
| --- | --- |
| Review rounds, commits pushed after the PR opened, recurring review comment themes | The check is too weak. Each repeated theme is a missing check. |
| CI failures after push, by job | Preflight or the local check is missing something. |
| Reverts, and fix PRs touching the same files within days | Defects that passed the check. |
| PRs closed unmerged, human commits on the agent's branch | Wrong direction. |
| Human comments that answer a question or redirect scope | A gap in the brief or a missing decision rule. |
| Time to first review, idle time waiting on a human | Waiting and relays. |
| PR size against review time and defect rate, merge conflict frequency | Tasks are sized or split badly. |

The analysis tags agent-authored PRs by task type, has a model classify review comments into themes, and ranks themes by frequency times cost. The top theme becomes the next check or rule. PR data shows only what reached GitHub. Mid-task stops, tool errors, and wasted paths live in agent session traces, so the two are read together.

## Open questions

- Should the human see a brief before dispatch always, or only when its assumptions exceed some risk level?
- What is the right budget before a stuck loop escalates?

## Outcome

The human's job becomes three things: writing checks, designing loops, and answering the few questions that need a human.
