---
name: auto-develop
description: 'Develop a feature, bug fix, or improvement end to end, either from a Jira ticket or from a description the user gives. The workflow fetches the ticket (or asks the user what to implement when there is none), clarifies unclear requirements with the user, validates scope, plans the smallest necessary changes, implements them, proves them with observed evidence, and then runs the code-reviewer skill and addresses its findings. Large changes may be split into stacked PRs only after user confirmation. Use when the user says "implement JIRA-123", "work on this ticket", "fix this bug autonomously", "develop this feature", or "pick up <ticket>".'
argument-hint: 'Jira key or URL (e.g. PROJ-1234), or a description of the work; optionally repo/branch hints'
---

# Auto Develop

Take a Jira ticket, or a task the user describes, from understanding to reviewed changes. **Do only what is necessary.** Do not refactor, rename, reformat, or "improve" code outside the ticket's scope.

## When to Use
- "Implement PROJ-123", "work on this ticket", "fix this bug", "develop this feature/improvement autonomously".

## Workflow

### 1. Get the Requirements
**If the user gave a Jira key or URL:**
- Use the Jira tools (search the tool list for `getJiraIssue`). Get the summary, description, acceptance criteria, issue type, priority, comments, linked or sub-issues, and remote links (`getJiraIssueRemoteIssueLinks`).
- Also read the linked issues or parent epic when they affect the requirements.

**If no Jira ticket was given:**
- If the user's message already describes the work clearly, use that description.
- Otherwise, use `ask_user` to ask what to implement. Ask for:
  - the type of work (feature, bug, or improvement),
  - the expected behavior, or the steps to reproduce for a bug,
  - the acceptance criteria,
  - the target repo or branch,
  - a related Jira key, if one exists.
- Treat the user's answers as the "ticket" for the rest of this workflow.

### 2. Understand What Needs to Be Done
- Restate the goal in 2–4 sentences: the problem, the expected behavior, and the acceptance criteria.
- **If anything is ambiguous, ask the user with `ask_user`.** Examples include missing acceptance criteria, conflicting comments, unclear expected behavior, or an unknown target repo or branch. Do not guess.

### 3. Determine and Validate Scope
- Find the relevant code with code search, grep, and view. Trace the actual code path; do not assume it.
- For bugs, reproduce the bug or confirm the root cause in the code before you plan a fix. Keep the reproduction, because step 6 reuses it as proof.
- **Check for existing work.** After `git fetch origin`, search for open or merged PRs and recent base-branch commits that already solve this or compete with it, for example `gh pr list --state all --search "<KEY>"` and `git log origin/<base> --oneline --grep "<KEY>"`. If the work is already done or superseded, stop and tell the user.
- Write down what is **in scope** and what is **out of scope**.
- Check the scope against the ticket. If the work is clearly bigger or smaller than the ticket implies, tell the user and confirm before you continue.

### 4. Plan the Changes
- List each file to change, what will change, and why. Map every change to a requirement or acceptance criterion.
- List the tests you will add or update. Add tests only for behavior that this change introduces or fixes.
- **Recheck the plan:** remove any change that does not trace back to the ticket. Confirm that the plan covers every acceptance criterion.
- **Check the size:** if the change is large (many files or subsystems, or hard to review as one PR), propose splitting it into **stacked PRs** with a clear order and purpose for each one. **Get user confirmation before you split.**
- **Hard decisions:** if the approach is ambiguous or high-stakes (several viable designs, an unclear root cause, or a scope boundary that is hard to call), use the `council` skill to reach a decision. Present that decision when you check in with the user.

### 5. Implement
- Run `git status --short` first. Treat existing uncommitted changes as the user's: don't stash, discard, or mix them into this change. If they overlap the files you need, ask.
- Run `git fetch origin`, then create a branch named with the ticket key from the latest base, for example `PROJ-123-short-desc` from `origin/<base>`. Never work directly on the default branch.
- Make surgical changes that follow the repo's existing conventions. Fix the case that happened, not every case that could. Prefer the fix that deletes or narrows code over adding a new helper, option, cache, or lock.
- Keep existing comments; when simplifying them, keep the rationale.
- Run the smallest relevant build, lint, and test commands, and fix any failures caused by the change.

### 6. Prove It
Proof is evidence you saw yourself in this session, such as command output, a reproduction, or a screenshot. Use the most direct proof the change allows:
- **Bugs:** rerun the step 3 reproduction and watch it pass now.
- **New behavior:** if no test covers it and the repo has a test setup, add one focused test that fails without the change.
- **Checks:** run the checks the repo's CI runs for the touched area (typecheck, lint, tests). Tell pre-existing failures apart from new ones.
- **UI or runtime behavior:** run the app and look at the result. Passing mocked tests do not prove the user experience works.

If you can't prove a change, say so and ask the user. Don't claim it works. Record `git diff --shortstat origin/<base>` as the pre-review diff size.

### 7. Review with code-reviewer
- Invoke the `code-reviewer` skill on the change (or on each stacked PR separately). Pass it the intended scope, the out-of-scope list, and the check results from step 6. For high-risk changes, or when the user asks, use its council mode.
- Check each finding yourself before you act on it:
  - **Agree:** make the smallest fix, then prove it again.
  - **Disagree:** reply with evidence (file:line, a test, or output). Don't agree just to end the loop.
  - **Out of scope or minor** (another code path, a rare edge case, a small performance cost, extra compatibility): decline in one line and list it as a follow-up.
- Review loops tend to grow the diff. Compare `git diff --shortstat` with the pre-review size. If it grew, check whether every addition is needed and cut what isn't.
- Rerun validation after fixes. Re-review only the changed code and open findings, within code-reviewer's round budget.

### 8. Report
- Summarize the ticket goal, the changes made with file links, the **proof** for each change, the diff size before and after review, the review findings with how each was resolved, any decisions you made on the user's behalf, and follow-ups or open questions.
- Do not commit, push, or open PRs unless the user asks. Ask the user whether to proceed.
- **If the user asks for a PR:** first check that no PR already exists for the branch. Follow the repo's title convention, and write a title that says why the change matters, not just what changed. Open the description with the problem in plain words, then explain the fix briefly. Don't lead with a list of changed files. Add one line on how you proved the change. Include the review or council summary. End with `Created with <model> in <harness>.`, using the model name the harness reports. Before submitting, check the title and description for secrets or private details.

## Guardrails
| Do | Don't |
|---|---|
| Ask when requirements are unclear | Invent requirements or acceptance criteria |
| Change only what the ticket needs | Refactor or reformat unrelated code |
| Confirm before splitting into stacked PRs | Split or bundle PRs without asking |
| Prove each change with evidence you observed | Declare the work done because it compiles or mocked tests pass |
| Push back on review findings with evidence | Let review rounds grow the diff one "cheap" fix at a time |
| Leave the user's uncommitted work alone | Stash, discard, or bundle unrelated local changes |

## Example
**Ticket:** "PROJ-42: Login fails when the email has uppercase letters."
- ✅ Normalize the email to lowercase in the login lookup, and add a test with a mixed-case email.
- ❌ Also rewrite the auth module, rename variables, and update unrelated validation.
