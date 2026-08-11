# iOS Agents for Claude Code

Reusable Claude Code subagents for production-focused iOS development,
ported from [custom-codex-agents](https://github.com/Megharth/custom-codex-agents).

Four focused agents:

- `ios-expert` — implements and validates production-grade iOS changes.
- `code-review-expert` — reviews motivation, correctness, complexity, style,
  comments, maintainability, and safer simpler alternatives.
- `ui-frontend-expert` — advises on modern, minimal, accessible, responsive,
  and user-friendly UI layout and interaction design.
- `health-algorithm-expert` — advisory-only design partner for algorithms
  that analyze health/biometric data (heart rate, sleep, activity, HRV, and
  similar trends). Proposes the pipeline, edge cases, validation approach,
  and privacy/regulatory considerations; does not implement or touch real
  health data itself.
- `project-manager` — delegation-only orchestrator that turns a spec into
  parallel work items and drives each through scope, test-first,
  implementation, and review sub-agents, committing one commit per item. Never
  writes code itself; asks for an agent when no suitable one is available.

All agents favor concise, high-signal responses and minimal complete
changes.

## Install

Claude Code subagents are Markdown files with YAML frontmatter, discovered
from `agents/` directories.

### Project-scoped

From this repository, copy the agents into the project where you want to use
them:

```sh
mkdir -p .claude/agents
cp /path/to/this-repo/agents/*.md .claude/agents/
```

Resulting layout:

```text
your-app/
├── .claude/
│   └── agents/
│       ├── ios-expert.md
│       ├── code-review-expert.md
│       └── ui-frontend-expert.md
└── ...
```

### Personal (all projects)

```sh
mkdir -p ~/.claude/agents
cp /path/to/this-repo/agents/*.md ~/.claude/agents/
```

## Use

Run `claude` from your project and either let Claude pick the right subagent
automatically (based on each agent's `description`), or ask explicitly.

`ios-expert` is a focused executor: it writes iOS implementation code and
tests to a brief. Scoping, UI advice, and code review are separate agents (or
your own responsibility) — `ios-expert` does not delegate to them itself.

To implement a change:

```text
Use the ios-expert subagent to implement this feature. Brief: [motivation,
intended behavior, edge cases, files/components involved, test surface].
Make the smallest production-ready change, add focused tests where useful,
and run the relevant checks.
```

To write tests before an implementation exists:

```text
Use the ios-expert subagent in tests-only mode. Brief: [intended behavior
and edge cases]. Write tests only — do not add implementation. The tests are
expected to fail until the implementation lands.
```

To address review findings:

```text
Use the ios-expert subagent to fix these review findings: [findings].
Address only these findings and anything they directly break; do not take on
unrelated cleanup.
```

For a motivation-aware code review:

```text
Use the code-review-expert subagent to review this change. First understand
the motivation from the issue, PR description, tests, and surrounding code.
Then check correctness, regressions, edge cases, complexity, style, comments,
maintainability, and simpler approaches that preserve all functionality.
Return prioritized findings with file references and do not edit files.
```

If the motivation is not present in the current context, provide the issue
or PR description with the review request. The reviewer will otherwise infer
the intended behavior from the diff and surrounding code and clearly mark
any remaining uncertainty.

For UI advice without implementation:

```text
Use the ui-frontend-expert subagent to design this screen before writing
code. Understand the user goal, recommend the information hierarchy and
interactions, and check accessibility, Dynamic Type, localization, dark
mode, keyboard, responsive layouts, and loading/error/empty states. Return
advice only; do not edit files.
```

`ios-expert` no longer launches other subagents itself. Ask
`ui-frontend-expert` for design advice first when you want it, then pass the
resulting brief into `ios-expert`. Follow up with `code-review-expert`
(directly or via `project-manager`) for the review.

## Composing the agents

The recommended flow for a non-trivial iOS change is:

1. (Optional, for UI work) Run `ui-frontend-expert` read-only for layout,
   interaction, accessibility, responsive, and localization recommendations.
2. Run `ios-expert` with a brief that includes the motivation, intended
   behavior, edge cases, files/components involved, and (when known) the test
   surface — plus any advice from step 1.
3. Run `code-review-expert` on the resulting diff. If it reports blocking
   findings, hand them back to `ios-expert` in fix mode.

`project-manager` automates exactly this composition across many work items
in one spec (see below).

Use `/agents` in the Claude Code CLI to list, inspect, or edit installed
subagents. Claude can also delegate to these subagents automatically when a
task matches an agent's description.

## Spec-driven delegation with `project-manager`

`project-manager` is a pure orchestrator. Given a spec file it decomposes the
work into items, builds a dependency graph, and runs independent, non-
overlapping items in parallel. For each item it drives a fixed pipeline
entirely through sub-agents: a read-only expert scopes the work, a test agent
writes tests first, an implementation agent makes them pass, and a
`code-review-expert` reviews the change. Blocking findings trigger a fix agent,
and the review→fix loop runs at most twice. Each work item lands as a single
commit containing its tests, implementation, and fixes.

It never writes, edits, or fixes code itself — not even trivial items. If no
available agent fits a required role (scope, tests, implementation, review, or
fix), it stops that item and asks you to provide the missing agent rather than
doing the work.

```text
Use the project-manager subagent. Read specs/feature.md, decompose it into
work items, delegate each to the appropriate specialist agents, and commit one
commit per item.
```

Two operational notes:

- **Invoke it directly.** It needs the Task tool to spawn sub-agents, and
  sub-agents generally cannot spawn further sub-agents. If launched as another
  agent's sub-agent, it stops and reports that it must be invoked at the top
  level rather than doing the work itself.
- **Commits are serialized.** Parallelism is in the work, not the committing —
  only items with disjoint file sets run concurrently, and commits land one at
  a time on the current branch. Unresolved blocking findings after two review
  cycles are filed via `gh issue create` when `gh` is available, or surfaced in
  the final report otherwise.

## Model and cost guidance

Each agent defaults to the `sonnet` model. Edit the `model:` field in the
frontmatter (`sonnet`, `opus`, `haiku`, or `inherit`) to change this per
agent. Subagents consume additional tokens because each launched agent
performs its own model and tool work, so a normal single-agent request is
cheaper for simple changes. For high-risk work, consider `opus`.

## Scope

These agents provide engineering guidance and implementation support. They
do not guarantee bug-free code; they are expected to report verification
results and remaining uncertainty. The consuming project remains responsible
for its deployment target, product decisions, signing, secrets, compliance,
and final release validation.

## License

Add the license appropriate for your distribution before publishing
publicly.
