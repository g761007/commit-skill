# commit

A [Claude Code](https://docs.claude.com/en/docs/claude-code) skill that analyzes
your current git working-tree changes and produces copy-pastable commit message
suggestions that match the repository's own commit convention.

It reads the repo's recent history to detect the convention in use (Conventional
Commits, plain imperative, emoji prefixes, language, scope vocabulary), groups
unrelated changes into separate atomic commits, and — by default — only
*suggests* messages. It never commits unless you explicitly ask it to.

## What it does

- **Analyzes the working tree** — staged, unstaged, and untracked changes.
- **Detects your convention** — infers format, emoji usage, language, and scopes
  from `git log`, then mirrors the dominant pattern instead of imposing one.
- **Groups into atomic commits** — unrelated changes are split into separate
  commit suggestions, each with the `git add` needed to stage it.
- **Suggests by default, commits on request** — committing is hard to undo in a
  shared history, so the skill prints messages for you to review. It only runs
  `git commit` when you clearly ask (e.g. "直接 commit", "commit 吧", "go ahead and commit").
- **Never pushes** unless explicitly told to.

## Installation

### Using the `skills` CLI (recommended)

The [skills](https://github.com/vercel-labs/skills) CLI installs this skill into
Claude Code (and many other agents) with one command:

```bash
# Install to the current project
npx skills add g761007/commit-skill

# Install globally, targeting Claude Code
npx skills add g761007/commit-skill -g -a claude-code

# List what's in the repo without installing
npx skills add g761007/commit-skill --list
```

### Manual install

Clone this repository into your Claude Code skills directory so the folder is
named `commit`:

```bash
git clone https://github.com/g761007/commit-skill.git ~/.claude/skills/commit
```

Or copy just the skill file into an existing setup:

```bash
mkdir -p ~/.claude/skills/commit
cp SKILL.md ~/.claude/skills/commit/SKILL.md
```

## Usage

In any git repository, invoke the skill directly:

```
/commit
```

With no arguments it immediately inspects the current changes and prints commit
suggestions — it won't ask what you want first.

To have it perform the commit for you, include an instruction:

```
/commit 直接 commit
```

You can also just describe what you want in plain language ("幫我看一下這次改了什麼",
"write me a commit message", "commit 吧") and the skill triggers on its own.

## Example

For a repo whose history uses Conventional Commits + emoji + Traditional Chinese,
a change that adds a dark-mode toggle yields:

```
feat(settings): ✨ 新增深色模式切換
```

For a repo whose history uses plain English imperative messages, the same skill
produces:

```
Fix null token crash on session refresh
```

The output always matches the convention it detects in *your* repo's history.

## License

[MIT](LICENSE)
