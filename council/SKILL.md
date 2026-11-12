---
name: council
description: 'Have a council of three models from different providers (Claude Opus, GPT Sol, Gemini) independently assess a development question, such as a code review, design or approach choice, scope decision, bug root cause, or issue triage, and reconcile their answers into one final decision. Use when the user says "council", "ask the council", or "get multiple opinions", or when the code-reviewer or auto-develop skill needs a high-stakes or ambiguous decision.'
argument-hint: 'The question/decision, or a review target (PR, branch, local changes)'
---

# Council

Three delegates, each a strong model from a different provider, independently assess the same question. You act as the chair: you verify their claims against the code, settle disagreements, and give **one final decision**.

## When to Use
- The user says "council", "ask the council", or "get multiple opinions".
- From the `code-reviewer` skill: when the user asks for a council review, or when the change is high-risk (security, data migration, public API, concurrency).
- From the `auto-develop` skill: for ambiguous or high-stakes decisions, such as choosing between implementation approaches, deciding whether something is in scope, confirming a bug's root cause, or deciding whether to split work into stacked PRs.
- Skip it for trivial questions, because a council is expensive.

## Members
If the user names council members, use those. Otherwise, pick the newest flagship model from each provider in the `task` tool's model list, and set it through the `model` parameter. The models below are only examples. Re-check the list rather than hardcoding them:

| Seat | Example model | Reasoning effort |
|---|---|---|
| Anthropic | `claude-opus-5.5` | `medium` |
| OpenAI | `gpt-6.1-sol` | `medium` |
| Google | newest available Gemini (e.g. `gemini-3.8-flash`; prefer a Pro model if one is listed) | `medium` if supported, otherwise the default |

Keep reasoning effort at `medium` or lower. Never use `high` or above unless the user explicitly asks.

If a provider or a named member is unavailable, **report it to the user instead of quietly substituting** another model.

## Procedure

### 1. Prepare first
- Before you ask for a review, fix known issues, address outstanding PR comments, and run the relevant checks. Give the council the **current** implementation; don't hold back known fixes until the reviews come back.
- If the user wants to discard the implementation and explore new ideas, skip fixes to code that will be replaced.
- Write a short **decision brief**. Delegates get no chat history, so the brief must stand on its own:
  ```
  Read only. Do not edit files, commit, push, comment on PRs, or start subagents.
  Question: <the decision or review target>
  Repo: <abs path>   Base: <sha>   Diff: <exact command, incl. untracked files>   (reviews only)
  Intent: <scope>   Out of scope: <non-goals>   Constraints: <...>
  Context: <Jira/PR summary, key files, check results, prior threads>
  Options: <A / B / ... or "open">
  Hold to the smallest change that meets the intent. A bug on another path, a rare edge case,
  a small perf cost, or extra compat is minor. Flag code, tests, or guards the intent does not need.
  Reply in the format below.
  ```

### 2. Convene in parallel
- Start all three delegates in **background** mode with the same brief.
  - Use the `code-review` agent type for reviews.
  - Use `general-purpose` or `rubber-duck` for design and decision questions.
- Delegates are **read-only**. Tell them not to edit files.
- Ask each delegate to return:
  ```
  Recommendation: <option / verdict: APPROVE | CHANGES>
  Confidence: <low|medium|high>
  Score: <N/10>            # code reviews only
  Key findings: <id> [critical|major|minor] <file:line> <claim>. Evidence: <...>
  Risks / objections: <...>
  Out-of-scope suggestions: <listed separately>
  ```
- A timeout, an error, or no reply is **not** agreement. Record that delegate as `no response`. Restart it once at most.

### 3. Don't wait idle
- When a result arrives, immediately check its claims against the code while the other delegates keep running.
- **Don't edit files while any delegate is still reviewing the same checkout.**
- If the user marked a delegate as non-blocking, don't wait for it. Treat a late result as feedback and don't record its score.

### 4. Deliberate and decide
- **All three agree:** adopt the shared recommendation after a quick check.
- **Two of three agree:** usually go with the majority. Before deciding, check the dissent's evidence yourself, because one valid objection with concrete evidence outweighs two unsupported votes.
- **No agreement:** compare the evidence, decide yourself, and explain why. If the trade-off is really a product or business choice, ask the user with `ask_user`.
- Drop any claim that you can't back with code or with evidence you verified. Recalibrate severity against the intent, whatever label a delegate gave it.
- Merge duplicate findings from different delegates into one ledger with stable IDs (`C1`, `C2`, …). Track each one as `id | severity | raised by | status (open / fixed / rejected / declined) | evidence`.
- Present one **final decision** with its reasons and the dissent that was overruled.
- **When another skill calls the council** (`code-reviewer` or `auto-develop`), return the decision and ledger to that skill. If the trade-off isn't a product or business choice, don't stop to ask the user.

### 5. Iterate (code reviews)
- For each finding, either agree and make the smallest fix (prefer deleting or narrowing code), or disagree and give evidence. Don't agree just to end the loop.
- After fixes, convene a **fresh round**. Each full review pass counts as one round. Later rounds get the brief, the ledger (`fixed:`, `disagree:`, or `declined:`, each with evidence), and the change since the last round. Tell delegates to check only the open IDs and the changed code, and to mark each old ID `resolved` or `still open: <reason>`.
- **Budget:** 3 rounds by default. Extend to 5 only for security, auth, data loss, or concurrency while a Critical or Major finding is still open with evidence, or when the user asks.
- **Stop** when all delegates approve the current diff and no Critical or Major finding is still open, or when a round returns only minor findings. If an item is still disputed after two exchanges, show both sides briefly and let the user decide. Running out of budget is **not** approval; report what is still unresolved.
- **Fight scope creep.** Repeated reviews tend to suggest "nice-to-haves". Fix only what the stated goal needs, and record anything else as a follow-up. If the diff grew during the rounds (`git diff --shortstat`), cut back before the next round.

## Output

```
## Council decision: <question>
**Members:** Claude Opus (<model>), GPT Sol (<model>), Gemini (<model>)
**Decision:** <one final decision>
**Why:** <2–4 sentences, evidence-based>
**Agreement:** <unanimous | 2–1 (dissent: X — reason overruled/accepted) | split — chair decided>
**Key findings:** <verified items with file:line>
**Follow-ups (out of scope):** <...>
```

### Code review scores
For a code review, record each delegate's score out of 10 for every round. If a delegate gives no score, write `no score`; never invent one.

```markdown
## Council review

Rounds: 2

| Delegate | Round 1 | Round 2 |
| --- | --- | --- |
| Claude Opus | 7/10 | 9/10 |
| GPT Sol | 8/10 | 10/10 |
| Gemini | 7/10 | 9/10 |
```

- If the work ends in a PR, put this section in the PR description, before the model and harness line at the end. If the PR is already open, **edit the description**; don't post a comment for it.
- If there is no PR, include the section in the final reply only.

## Anti-patterns
- Running the delegates one after another when they could run in parallel.
- Showing the user three separate reports instead of one decision.
- Accepting the majority without checking the dissent's evidence.
- Letting council rounds expand the scope.
- Quietly substituting a model when a council member is unavailable.
- Counting a silent or timed-out delegate as a vote.
- Re-raising a declined or fixed finding under a new ID in a later round.
