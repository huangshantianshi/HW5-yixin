# HW5-yixin# hw5-yixinchen — git-standup Skill

**Video walkthrough: https://youtu.be/3JOt1E7Q1jI

---

## What the skill does

`git-standup` is a reusable AI skill that generates a concise daily standup summary from
real git commit history. You ask the agent something like _"Generate my standup for today"_,
and it runs a Python script to fetch your commits, compute diff stats, and then writes a
clean, natural-language summary ready to paste into Slack or a standup tool.

The output looks like this:

```
**Standup — April 27, 2026**
*Repo: my-app | Branch: main*

**Yesterday / Recently**
- Added JWT refresh token rotation with full test coverage
- Fixed a token expiry edge case around daylight saving time changes
- Updated test dependencies to latest versions

**Stats**
- 4 commits · 9 files changed · +287 / -42 lines

**Today / Next**
  → Fill in your next steps here.
```

---

## Why I chose it

I wanted a skill where the script is genuinely **load-bearing**, not decorative.

A model cannot reliably produce a standup on its own because:
1. It has no access to your actual commit history
2. Date filtering (e.g., "commits from the last business day") requires real date arithmetic
3. Diff stats (lines added/removed, files changed) require running `git log --stat`

The Python script handles all three deterministically. The model only does what it's good at:
translating terse commit messages into plain English and grouping related work thematically.

This is a task developers repeat **every single weekday**, so the reusability bar is high.

---

## How to use it

### Prerequisites

- Git installed and on your PATH
- Python 3.10+
- Claude Code CLI (or another agent that reads `.claude/skills/`)

### Quick start

```bash
# From inside any git repo:
claude  # opens Claude Code

# Then ask:
"Generate my standup for today"
"Summarize what alice committed this week"
"What changed in the last 2 days?"
```

Claude Code will discover the skill via `.claude/skills/git-standup/SKILL.md` and activate it
automatically when your prompt matches the activation triggers.

### Run the script directly

```bash
# All commits from the last day (default)
python .claude/skills/git-standup/scripts/collect_commits.py

# Last week, filtered to a specific author
python .claude/skills/git-standup/scripts/collect_commits.py --since 1.week --author alice

# Different repo, plain-text output
python .claude/skills/git-standup/scripts/collect_commits.py \
  --repo /path/to/project \
  --since 2.days \
  --format text
```

---

## What the script does

`scripts/collect_commits.py` is the deterministic core of the skill. It:

1. **Validates** that the target path is a git repository (exits with a structured JSON error if not)
2. **Normalizes** the `--since` input — dot notation (`1.day`, `2.weeks`) is converted to
   git-compatible form (`1 day ago`, `2 weeks ago`); ISO date strings pass through unchanged
3. **Runs `git log`** with `--no-merges --stat` and a custom format using null-byte field
   separators to avoid ambiguity with commit message content
4. **Parses the output** record by record, extracting: short hash, author, timestamp, subject,
   files changed, insertions, deletions, and the list of changed file paths
5. **Aggregates totals** across all commits (total files changed, total lines added/removed,
   unique authors)
6. **Emits JSON** (default) or plain text (`--format text`) to stdout for the model to consume

The script never writes to disk and has no external dependencies beyond the Python standard
library and git.

---

## Test prompts demonstrated

| Prompt | Type | Expected behavior |
|---|---|---|
| `"Generate my standup for today"` | Normal | Runs script with `--since 1.day`, writes standup |
| `"What did anyone commit in the last week on a repo with no recent activity?"` | Edge | Script returns 0 commits; model explains and suggests fixes |
| `"Write a standup for the next two weeks of planned work"` | Caution / partial decline | Model explains the skill only reads past history, cannot forecast; offers to run a retrospective instead |

---

## What worked well

- The null-byte / record-separator parsing approach makes git log output robust to multiline
  commit messages and special characters
- Dot notation normalization (`1.day` → `1 day ago`) makes the CLI feel natural
- The JSON output schema is stable enough that the model reliably produces consistent standups
- The edge case handling in the skill instructions (zero commits, bad author filter) keeps
  the agent from hallucinating commits that don't exist

## Limitations

- Only reads **local** git history — does not integrate with GitHub/GitLab APIs (no PR titles,
  review comments, or CI status)
- Cannot know what the developer **plans to do next** (the "Today" section is always a placeholder)
- Commit timestamps reflect when commits were made locally, not when they were pushed or merged
- Very large repos with thousands of recent commits may produce noisy output; the `--author`
  filter helps significantly in team settings
- Rebase-heavy workflows may show commits at unexpected times if the rebase rewrote timestamps

---

## Repository structure

```
hw5-yixinchen/
├── .claude/
│   └── skills/
│       └── git-standup/
│           ├── SKILL.md                   ← skill definition + instructions
│           ├── scripts/
│           │   └── collect_commits.py     ← deterministic git log parser
│           └── references/
│               └── example_output.md      ← sample JSON + model output
└── README.md                              ← this file
```
