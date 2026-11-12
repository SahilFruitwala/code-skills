---
name: code-reviewer
description: 'Context-aware, high-signal code review of local changes, a branch diff, or a GitHub PR (including stacked PRs). Verifies the implementation matches the intended scope, flags scope creep, and reports findings by severity. Runs a review loop: Claude Opus reviews first, then GPT Sol revalidates. Use when the user says "review", "code review", "review my changes", "review this PR", or "check this branch".'
argument-hint: 'PR URL/number, base branch, or nothing for local changes'
---

# Code Reviewer (Context-Aware, Two-Model Loop)

Review a change to find real bugs and risks. Check that the change does what it is supposed to do, and nothing more. Report only high-signal findings, ordered by severity.

## When to Use
- "Review my changes", "review this branch", "code review", "review PR #123 / <url>".
- Before opening or merging a PR, or when a stacked PR needs to be checked on its own.

## Review Targets
Run `git fetch origin` first so the base is current. Pick **one** diff command and use it for every pass and round.

| Target | How to get the diff |
|---|---|
| Local changes | `git --no-pager diff HEAD` (staged + unstaged), plus `git status --short` to find **untracked** files. `git diff` does not show untracked files, so read them directly. |
| Branch vs base | `base=$(git merge-base HEAD origin/<base>)`, then `git --no-pager diff $base` (add `HEAD` to exclude uncommitted work) |
| GitHub PR | `gh pr view <n> --json title,body,baseRefName,headRefName,headRefOid,files,commits,url,comments,reviews`, `gh pr diff <n>`, and `gh pr checks <n>` |

Don't stash, discard, or check out over the user's uncommitted work. If a PR review needs a checkout and the tree is dirty, use `gh pr diff` or a separate `git worktree` instead.

## Procedure

### 1. Understand the context first (do not skip)
Before judging any code, figure out what the change is *supposed* to do:
- Read the PR title and description, the commit messages, and any linked issue or Jira ticket (use the Jira tools if a key like `ABC-123` appears).
- Read the surrounding code for every changed hunk, including callers, callees, related tests, and conventions in nearby files. Don't review a hunk without this context.
- Write down the **intended scope** in 1–3 sentences and the **out-of-scope** items in one line. Every scope and severity check below compares against these statements.
- Note the **check status**: `gh pr checks <n>` for PRs, or the results the caller passed in. Mention missing or already-failing checks in the reviewer brief.

#### For PRs: read the PR body and existing discussion
- **PR body**: Treat it as the main statement of intent. Use its stated goals, non-goals, testing notes, and any "intentionally changed" or "follow-up" items to define scope. Flag anything the body claims that the diff doesn't implement.
- **Existing comments**: Collect all three kinds:
  - General PR comments and review summaries: `gh pr view <n> --json comments,reviews`
  - Inline review comments: `gh api repos/{owner}/{repo}/pulls/<n>/comments --paginate`
  - Thread resolution state (resolved/outdated):
    ```
    gh api graphql -f query='query($o:String!,$r:String!,$n:Int!){repository(owner:$o,name:$r){pullRequest(number:$n){reviewThreads(first:100){nodes{isResolved isOutdated path line comments(first:20){nodes{author{login} body}}}}}}}' -F o=<owner> -F r=<repo> -F n=<n>
    ```
- Use the existing discussion like this:
  - **Don't duplicate.** Don't re-raise a point a reviewer already made. If it is still unresolved and still applies, list it under **Open from prior review** and add any new evidence.
  - **Verify claimed fixes.** For each thread that is resolved or answered with "fixed" or "done", check that the current code actually addresses it. Report any that don't as **Prior comment not addressed**.
  - **Respect agreed decisions.** If the author and reviewers agreed in a thread to defer something, keep it out of scope, or accept a trade-off, don't flag it again. Mention it only if the code now contradicts that agreement.
  - **Skip outdated threads** unless the issue they raised still exists in the new code.

### 2. Detect stacked PRs and isolate the real delta
A PR is likely stacked when its base branch is not the default branch (`main`, `master`, or `develop`), or when its base branch is the head branch of another open PR.
- Check with: `gh pr view <n> --json baseRefName` and `gh pr list --head <baseRefName> --json number,title,url`.
- If it is stacked, **review only this PR's delta against its parent branch** (`git diff <parentBranch>...<thisBranch>`). Don't flag issues that come from the parent PR. If a parent change matters to this PR, mention it once as context and point to the parent PR.
- Note whether a parent PR's unmerged changes affect this PR, for example by changing an API that this PR relies on.
- For local or branch reviews, check `git log --oneline <base>..HEAD` for commits that belong to a different logical change.

### 3. Verify scope vs implementation
- **Completeness**: Does the implementation fully cover the intended scope? Flag missing pieces, such as unhandled cases, missing migrations, config, docs, or tests for new behavior.
- **Correctness of intent**: Does the code actually do what the description says? Flag mismatches.
- **Scope creep**: Flag changes that are not needed for the stated goal, such as unrelated refactors, drive-by renames, reformatting, dependency bumps, extra features, or speculative abstractions. **Exception**: skip the flag if the PR description, ticket, or user explicitly calls out that change as intended.
- **Over-engineering**: Flag new layers, options, or generalizations that the change doesn't need.

### 4. Review the code (high-signal only)
Look for these issues, and report only ones that are real and actionable:
- **Correctness / logic**: wrong conditions, off-by-one errors, null/undefined handling, race conditions, wrong state transitions, broken invariants.
- **Security**: injection, authn/authz gaps, secrets in code, unsafe deserialization, path traversal, missing input validation, sensitive data in logs.
- **Performance**: N+1 queries, unbounded loops or memory use, blocking calls on hot paths, missing pagination, needless repeated work.
- **Error handling & edge cases**: swallowed errors, missing cleanup or rollback, empty/large/concurrent inputs, partial failure.
- **Tests**: new or fixed behavior that no test covers, tests that would pass even without the change, brittle or over-mocked tests.
- **Readability & naming**: only when it could realistically cause misuse or bugs.
- **Architecture / design**: layering violations, wrong ownership, tight coupling, breaking public API or contract changes.
- **Comments & docs**: flag stale or misleading comments. Also flag removed comments that explained *purpose*. Trimmed comments should keep a concise rationale summary.

Do **not** report: pure style nits, formatting, personal preferences, or anything a linter or formatter already enforces.

**Calibrate severity against the intent.** A bug in the intended case is Critical or Major. A bug on another code path, a rare edge case, a small performance cost, or extra compatibility is Minor at most, whatever a reviewer labeled it. Flag code, tests, or guards that the intent doesn't need. The smallest correct change wins.

### 5. Two-model review loop
Run the review in passes with different models so each pass checks the other's findings. Pick the newest Claude Opus and GPT Sol from the `task` tool's model list (e.g. `claude-opus-5.5`, `gpt-6.1-sol`) rather than trusting the examples here. Set `reasoning_effort` to `medium` or lower. Never use `high` or above unless the user explicitly asks.

**Reviewer brief.** Every pass gets the same brief:
```
Review only. Do not edit files, commit, push, comment on PRs, or start subagents.
Repo: <abs path>   Base: <sha>   Diff: <exact diff command>   Untracked: <files or none>
Intent: <scope>   Out of scope: <non-goals>   Checks: <results / missing / pre-failing>
Stacked: <no | parent #N — review delta only>   Prior threads: <summary or none>
Read the surrounding code, not just the hunks. Look for bugs, broken callers, and mismatch with intent.
Use the checklist: <step 4 list>. Skip style unless it hides a bug.
Severity: critical/major = intended case wrong or real bug callers will hit; minor = everything else.
Reply:
VERDICT: APPROVE | CHANGES
F<n> [critical|major|minor] <file>:<line> <claim>. Evidence: <why>. Fix: <suggestion>
```

**Ledger.** Give each finding a stable ID (`F1`, `F2`, …) and keep those IDs across passes and rounds. Track each one as `id | severity | status (open / fixed / rejected / declined) | evidence`. This prevents the same finding from being re-raised under a new name.

1. **Pass A — Claude Opus (primary review).** Launch a `code-review` subagent with the brief.
2. **Pass B — GPT Sol (revalidation).** Launch a second subagent with the brief **plus the ledger from Pass A**. Ask it to:
   - confirm or reject each ID, with evidence (a file:line reference and its reasoning),
   - correct the severity where it's wrong,
   - add any high-signal issues that Pass A missed, using new IDs.
3. **Reconcile.** Check every disputed item and every Critical or Major finding against the code yourself. Keep findings that are backed by evidence and drop the rest. Agreement between the models is not proof. Recalibrate severity using the rule in step 4.
4. **Loop.** Run another Opus → Sol round only on the new or disputed IDs, or on code that changed since the last round. Tell the reviewers to mark each old ID `resolved` or `still open: <reason>`.
   - **Budget:** 3 rounds by default. Extend to 5 only for security, auth or permissions, data loss, or concurrency, while a Critical or Major finding is still open with evidence, or when the user asks.
   - **Stop early** when a round returns only Minor findings, or when the only fixes were to comments, docs, or formatting.
   - **Stuck:** if a finding is still disputed after two exchanges, show both sides in 2–3 lines each and let the user decide.
   - A timeout, an error, no reply, or running out of budget is **not** approval.

If a requested model is unavailable, use the closest model from the same family and say so in the report.

**Council mode:** If the user asks for a "council" review, or the change is high-risk (security, data migration, public API, concurrency), use the `council` skill instead of this two-model loop. Give it the context from steps 1–4. In the report, use its reconciled decision and score table.

### 6. Report
Present the final, reconciled findings and wait for the user. Stay read-only: don't apply fixes unless the user asks.
- **When another skill calls this one** (for example `auto-develop`), return the report and the ledger to that skill instead of waiting for the user. The caller fixes the findings and asks for a re-review of the changed code and open IDs.
- **Posting to GitHub:** only when the user asks. Start each comment with the attribution note below, lead with the conclusion, cite the code, and check the text for secrets. After posting, confirm that GitHub returned a URL. Resolve a thread only after your reply has posted.
  ```md
  > [!NOTE]
  > 🤖 **<model> responding on behalf of <user>**

  <comment>
  ```

## Output Format

```
## Review: <PR title / branch / "local changes">
**Intended scope:** <1–3 sentences>
**Stacked PR:** <No | Yes — parent: #123 (<branch>); reviewed delta only>
**Checks:** <green | failing: X (pre-existing?) | not run>
**Review loop:** <N> round(s) — Opus → Sol (<models used>); stopped because <approved | only minor left | budget | stuck>

### 🔴 Critical
1. **F1 <short title>** — `path/to/file.ts:42`
   <what is wrong and why it matters (concrete failure scenario)>
   **Fix:** <specific suggestion>

### 🟠 Major
...

### 🟡 Minor
...

### 💬 Prior Review Comments (PRs only)
- **Open from prior review:** <thread summary> — `file:line` — <still applies because…>
- **Prior comment not addressed:** <thread marked fixed/resolved but code still has the issue>

### 📐 Scope
- **Out of scope:** <change> — `file:line` — <why it is not needed for the stated goal>
- **Missing from scope:** <expected piece not implemented>

### ✅ Verdict
<Ready to merge | Merge after fixing Critical/Major | Needs rework> — <one-line reason>
```

Leave out any severity section that has no findings. If nothing turns up, say so plainly. Don't invent findings.

**Severity guide**
- **Critical**: data loss or corruption, a security hole, a crash or outage, or a broken core flow.
- **Major**: incorrect behavior in realistic cases, a missing key test, a breaking API change, significant scope creep.
- **Minor**: an edge-case bug, a misleading comment or name, a small inefficiency, minor out-of-scope changes.

## Anti-patterns
- Reviewing the diff without reading the surrounding code or the PR or ticket intent.
- Flagging a parent PR's changes while reviewing a stacked child PR.
- Ignoring the PR body or existing review threads, and then re-raising points that were already discussed or agreed on.
- Trusting "resolved" or "fixed" on a thread without checking the current code.
- Calling an intentional change, one the description explicitly mentions, scope creep.
- Padding the report with nits to look thorough.
- Reporting a finding that the revalidation pass rejected and that you didn't verify yourself.
- Reviewing only `git diff` output and missing untracked files.
- Re-raising a declined or fixed finding under a new ID in a later round.
- Treating a timeout, a silent reviewer, or the end of the budget as approval.
