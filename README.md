# iOS Agents for Claude Code

Reusable Claude Code subagents for production-focused iOS development,
ported from [custom-codex-agents](https://github.com/Megharth/custom-codex-agents).

Three focused agents:

- `ios-expert` — implements and validates production-grade iOS changes.
- `code-review-expert` — reviews motivation, correctness, complexity, style,
  comments, maintainability, and safer simpler alternatives.
- `ui-frontend-expert` — advises on modern, minimal, accessible, responsive,
  and user-friendly UI layout and interaction design.

All three favor concise, high-signal responses and minimal complete changes.

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
automatically (based on each agent's `description`), or ask explicitly:

```text
Use the ios-expert subagent to implement this feature. Inspect the project
first, consider material edge cases, make the smallest production-ready
change, and run the relevant tests:

[describe the feature]
```

For a focused review:

```text
Use the ios-expert subagent to review this change for correctness,
concurrency, accessibility, security, performance, and missing tests. Return
only actionable findings with file references.
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

When `ios-expert` develops UI, it requests this advice automatically before
implementation.

## Automatic review after implementation

When `ios-expert` changes code, it automatically launches exactly one
`code-review-expert` subagent (via the Task tool) before completing. The
reviewer receives the change motivation, changed files, tests, and known
tradeoffs. It reviews read-only; `ios-expert` evaluates the findings, fixes
valid issues, reruns the relevant checks, and reports the review outcome.

Install both agent files together for this workflow. If `code-review-expert`
is not installed or cannot be launched, `ios-expert` performs a focused
self-review and reports that the delegated review was unavailable.

## Automatic UI consultation

When an `ios-expert` task includes UI or frontend work, it automatically
launches exactly one `ui-frontend-expert` subagent before implementation.
The UI agent is read-only and returns layout, interaction, accessibility,
responsive, and localization recommendations. `ios-expert` incorporates the
advice, implements the change, and then launches `code-review-expert` once
for the final code review.

Install all three agent files together for the complete workflow. If the UI
agent is unavailable, `ios-expert` performs a focused self-review and
reports that the delegated consultation could not run.

Use `/agents` in the Claude Code CLI to list, inspect, or edit installed
subagents. Claude can also delegate to these subagents automatically when a
task matches an agent's description.

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
