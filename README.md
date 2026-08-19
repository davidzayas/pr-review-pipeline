# pr-review-pipeline

A Claude Code plugin that reviews a queue of GitHub PRs in strict order: review → address Copilot comments → approve & merge, or request changes with an @mention to the author and wait for fixes before moving on.

## Prerequisites

- [GitHub CLI](https://cli.github.com/) installed and authenticated (`gh auth login`) with permission to review and merge in the target repo
- Run commands from inside a clone of the target repo

## Install (local)

Option A — direct path:

```
/plugin install /path/to/pr-review-pipeline
```

Option B — local marketplace (this repo's parent folder includes `marketplace.json`):

```
/plugin marketplace add /path/to/parent-folder
/plugin install pr-review-pipeline@local
```

Validate: `/plugin validate /path/to/pr-review-pipeline`

## Usage

```
/pr-review-pipeline:review 142 145 150
/pr-review-pipeline:review 142 145 --merge-method rebase --poll-interval 10 --max-wait 120
/pr-review-pipeline:status
/pr-review-pipeline:resume
```

## How it works

1. PRs are processed strictly in the order given.
2. Each PR is reviewed by the bundled `pr-reviewer` agent: correctness, security, tests/CI, and a disposition for every Copilot review comment.
3. **Approve** → posts an approval review with the reasoning, then merges (squash by default, `--delete-branch`).
4. **Request changes** → posts a changes-requested review with numbered, concrete fixes, @mentions and assigns the author.
5. A blocked PR is polled (default: every 5 min, up to 4 hours) for new commits or replies. On activity it is fully re-reviewed. The pipeline never advances past an unmerged PR.
6. If the wait budget runs out, state is saved to `.claude/pr-pipeline-state.json` and the run pauses — pick it up any time with `/pr-review-pipeline:resume`.

## Safety rails

- Never merges over failing required checks or branch protections
- Never pushes commits to the author's branch
- Pauses (rather than retries) on auth/permission errors

## Layout

```
pr-review-pipeline/
├── .claude-plugin/plugin.json
├── commands/
│   ├── review.md    # /pr-review-pipeline:review
│   ├── resume.md    # /pr-review-pipeline:resume
│   └── status.md    # /pr-review-pipeline:status
├── agents/
│   └── pr-reviewer.md
└── README.md
```
