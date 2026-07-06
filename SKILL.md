---
name: commit
description: >-
  Analyze the current git working-tree changes (modified, staged, and untracked
  files), group them into atomic units of work, and produce copy-pastable commit
  message suggestions that match the repository's own commit convention. Use this
  whenever the user has uncommitted changes and asks anything like "幫我 commit",
  "commit 訊息", "建議 commit", "write a commit message", "what should I commit",
  "幫我看一下這次改了什麼", or has just finished a code change and is ready to record
  it — even if they don't say the word "commit" explicitly. By default it only
  SUGGESTS messages and does not run git commit; only commit when the user clearly
  asks you to (e.g. "直接 commit", "commit 吧", "幫我提交").
---

# Commit

Turn the current uncommitted changes into well-scoped, convention-matching commit
message suggestions. The goal is to save the user the work of reviewing the diff
and phrasing each message, while respecting that the *decision* to commit is
usually theirs.

This skill orchestrates rather than executes directly: the actual git inspection
and message-drafting work happens in a subagent (via the Agent tool), not inline
in the main conversation.

## Default behavior when invoked with no arguments

This skill is normally invoked directly (e.g. the user types `/commit`) with no
further description. When that happens, **do not ask what the user wants
first** — just delegate immediately: spawn the subagent described below with
suggest-mode instructions. That is the whole point of invoking it — the user has
already told you what they want by calling it.

Only deviate from suggest mode if the invocation itself carries an instruction —
e.g. `/commit 直接 commit` or a surrounding message like "commit 吧". Pass that
instruction through to the subagent so it knows whether to actually commit.

## Delegate to a subagent

Do the actual work by calling the **Agent** tool once per invocation — don't run
the git inspection yourself in the main conversation.

- **subagent_type**: `general-purpose` (needs Bash to inspect git state, Read
  for untracked files, and Write for the handoff file below).
- **model**: default to the cheapest available model (currently `haiku`, i.e.
  Haiku 4.5). This task is mechanical — read a diff, match an existing
  convention, phrase a message — and doesn't need a stronger model by default.
  Only use a pricier model (e.g. `sonnet`) if the user explicitly asks for a
  smarter model, or if a cheap-model run comes back empty/unusable (see
  "Don't trust the chat reply" below) even after one retry.
- **prompt**: self-contained — the subagent starts with no memory of this
  conversation. Include, verbatim or paraphrased:
  - Whether this is **suggest mode** (default) or **commit mode** (only if the
    user explicitly asked to commit — phrases like "直接 commit", "commit 吧",
    "幫我提交", "go ahead and commit").
  - The full workflow from the "Workflow" section below.
  - The output format from the "Output format" section below.
  - Any extra instruction the user gave alongside the invocation (e.g. "only
    the auth files", "in English", "split by feature").
  - The **handoff file** instruction below.

### Don't trust the chat reply — use a handoff file

Cheap models reliably run the git commands but are unreliable about putting the
actual deliverable in their final chat message — in testing, Haiku often ended
with a bare "Done." / "Suggestion delivered above." instead of the real content,
even when explicitly told not to. Don't depend on the subagent's final message
for the deliverable. Instead:

1. Before spawning, pick a path in your scratchpad directory, e.g.
   `<scratchpad>/commit-suggestion.md`.
2. In the prompt, instruct the subagent to **Write** its full output (the exact
   "Output format" content below — or a plain "no changes" / clarifying-question
   message if that's what applies) to that exact path as its last action, in
   addition to whatever it says in chat.
3. After the Agent call returns, **Read that file yourself** — don't rely on
   what the subagent said back to you. If the file is missing, empty, or
   clearly doesn't contain the expected output, retry once (same model); if it
   fails again, retry with `sonnet`.
4. Relay the file's contents to the user per "After the subagent returns".

## Core principle: suggest, don't commit (by default)

The default mode is **suggest only**. The subagent analyzes the changes and
prints copy-pastable commit messages. It does **not** run `git add` or
`git commit` unless the user explicitly asked for it. This matters because
committing is hard to undo in a shared history, and the user often wants to
tweak wording, split differently, or stage selectively first. When in doubt,
suggest mode.

This also means the subagent must not create, edit, or delete *any* file in
the project — including things like adding a `.gitignore` to tidy up
untracked files it noticed. In testing, a cheap model did exactly this
unprompted, as an unrequested "fix" for clutter it spotted in `git status`.
Its only filesystem write, ever, is the handoff file described above; tell it
so explicitly in the prompt.

## Workflow (give this to the subagent)

### 1. Gather the changes

Run these together to see the full picture — staged, unstaged, and untracked:

```bash
git status --short
git diff            # unstaged changes
git diff --staged   # already-staged changes
```

For untracked files, look at their content directly (Read the file) so the
suggestion reflects what was actually added, not just the filename. If the diff
is very large, focus on the parts that reveal *intent* (new functions, changed
conditionals, deleted blocks) rather than reading every line.

If there are no changes at all, say so plainly and stop — don't invent a commit.

### 2. Detect the repository's commit convention

Never assume a format. Read what this repo actually does:

```bash
git log --oneline -20
```

Infer from the recent history:
- **Format**: Conventional Commits (`type(scope): desc`)? Plain imperative? Something else?
- **Emoji**: Are commits prefixed/infixed with emoji (e.g. `✨`, `🐛`, `♻️`)? Match the placement and the type→emoji mapping the repo uses.
- **Language**: Are descriptions in English, Traditional Chinese, etc.? Match it.
- **Scope vocabulary**: What scopes appear (e.g. `api`, `ui`, `auth`)? Reuse an existing scope when the change fits one.

Mirror the dominant pattern. If the history is mixed or empty, fall back to
Conventional Commits with a short imperative description in the language the user
is speaking, and say which convention you chose and why.

### 3. Group changes into atomic commits

A good commit is one logical change. Group the changed files/hunks by **intent**,
not by file count:

- Changes to the same feature/module that serve one purpose → one commit.
- Unrelated changes (e.g. a bugfix in `payments` and a refactor in `search`) → separate commits.
- A formatting-only or config change mixed in with logic → usually its own commit.

If everything is clearly one cohesive change, a single commit is correct — don't
split artificially. When you do split, briefly note which files belong to each
group so the user can stage them.

### 4. Produce the suggestions

For each group, output a copy-pastable commit message in a fenced code block,
matching the detected convention. Keep the subject line concise (roughly ≤ 50
chars of description after the type/scope). Add a body only when the *why* isn't
obvious from the subject — don't pad.

When you split into multiple commits, also give the matching `git add` for each
group so the user can stage and commit each one independently.

### 5. Commit only when asked

If the user explicitly asked to commit, stage the relevant files and commit each
group in order. Use a `git commit` with the message you proposed. After
committing, run `git log --oneline -n <count>` to confirm. If you committed
multiple atomic commits, list them back.

Never `git push` unless explicitly told to.

## Output format (suggest mode)

Use this structure so the result is easy to scan and copy:

```
## 變動分析

<1–2 sentences: what changed overall, and how you grouped it>

### Commit 1 — <files in this group>
\`\`\`
<commit message, convention-matched>
\`\`\`
（如需分開暫存）git add <files>

### Commit 2 — <files in this group>
\`\`\`
<commit message>
\`\`\`
```

If there's only one logical change, just show one commit block — skip the numbering.

## Examples

These show convention-matching, not fixed output — always match the *actual* repo.

**Example 1 — repo uses Conventional Commits + emoji + Traditional Chinese:**
Change: added a dark-mode toggle to the settings screen.
Output:
```
feat(settings): ✨ 新增深色模式切換
```

**Example 2 — repo uses plain English imperative:**
Change: fixed a null check in the auth token refresh path.
Output:
```
Fix null token crash on session refresh
```

**Example 3 — two unrelated changes in the working tree:**
A bugfix in the `payments` module and a tweak to a shared `logger` utility.
Suggest two separate commits, each scoped to its area, with a `git add` for each group.

## After the subagent returns

Read the handoff file (not the subagent's chat reply — see "Don't trust the
chat reply" above) and relay its contents back to the user **verbatim**. Don't
compress it into a "summary"; the exact text is what the user will copy. If the
file says there were no changes, or contains a clarifying question, pass that
along as-is too.
