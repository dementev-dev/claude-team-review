---
name: claude-team-review
description: >
  Adversarial code/plan review using Claude Code Agent Teams. Spawns
  a reviewer teammate that reads the project, runs tests, checks docs,
  and delivers findings. Lead fixes issues and requests re-review from
  the same teammate. Use when user says /claude-team-review, asks for
  team review, team-based code review, or wants an adversarial review
  without external dependencies.
user_invocable: true
---

# Claude Team Review

Spawns an adversarial reviewer **teammate** (Agent Teams) to review plans
or code. The lead fixes issues and requests re-review from the same
teammate. If the teammate is no longer active, the lead decides how
to proceed based on context. Maximum 5 rounds.

> **Requires:** Claude Code ≥ 2.1.32, experimental Agent Teams enabled
> (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in settings or environment).

---

## When to invoke

- `/claude-team-review` — auto-detect what to review
- `/claude-team-review plan` — force plan review
- `/claude-team-review code` — force code review
- `/claude-team-review <file-path>` — review a specific file (argument contains `/` or `.`)
- `/claude-team-review xhigh` — use max effort for the reviewer

## Instructions

### Step 1: Determine review mode

Check in priority order:

**1. Explicit argument** (`plan`, `code`, file path) → use it.
- For `plan` → skip all git checks, proceed to step 2.

**2. Claude Code Plan Mode** — if context contains the system message
"Plan mode is active" → mode = `plan`, skip git.

**3. Auto-detect** (no explicit argument, not in Plan Mode):

1. Check for code changes (any non-empty output means changes exist):
   - `git diff --name-only` — unstaged
   - `git diff --cached --name-only` — staged
2. Check if a plan exists in the current conversation context.

| Code changes? | Plan in context? | Mode               |
|---------------|------------------|---------------------|
| No            | Yes              | **plan**            |
| Yes           | Yes              | **code-vs-plan**    |
| Yes           | No               | **code**            |
| No            | No               | Ask the user        |

### Step 2: Spawn the reviewer teammate

Spawn a teammate using the `adversarial-reviewer` agent type.

Include in the spawn prompt a **briefing** with the review mode and
enough context to start. The reviewer is a full Claude Code session —
it will explore the repo, run git commands, and read files on its own.
Do not pre-collect diffs or file lists for it.

If spawning the teammate fails (Agent Teams not available), tell the user:
```
Agent Teams are not enabled. Add this to your settings.json or environment:
CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
Then restart Claude Code.
```

**For plan review:**

If the plan exists as a file:
```
You are reviewing an implementation plan.

Plan location: <path>

Review this plan with your full adversarial stance. Read the plan,
explore the project structure and relevant code to assess feasibility,
and deliver your findings.

End with VERDICT: APPROVED or VERDICT: REVISE.
```

If the plan is only in conversation context, include it inline:
```
You are reviewing an implementation plan.

<plan text>

Explore the project structure and relevant code to assess feasibility.
Deliver your findings.

End with VERDICT: APPROVED or VERDICT: REVISE.
```

**For code review:**
```
You are reviewing code changes in this repository.

Use git status, git diff, and any other git commands to find and
understand all changes. Read the surrounding code for context.
Run tests if available.

End with VERDICT: APPROVED or VERDICT: REVISE.
```

**For code-vs-plan review:**

Include the plan (path or inline text) and let the reviewer find
the code changes:
```
You are reviewing code changes against an implementation plan.

Plan: <path or inline text>

Use git to find all changes. Check: does the implementation cover
all plan steps? Where does it deviate? What is missing?

End with VERDICT: APPROVED or VERDICT: REVISE.
```

**Effort override:** if the user passed `xhigh`, use the appropriate
effort setting for the teammate.

### Step 3: Show findings

When the reviewer responds with findings:

1. Show the user the reviewer's response **verbatim** — do not rephrase:

```
## Team Review — Round N (mode: <plan|code|code-vs-plan>)

[Reviewer's response — verbatim]
```

2. Check the verdict:
   - **VERDICT: APPROVED** → proceed to Step 6 (Done)
   - **VERDICT: REVISE** → proceed to Step 4 (Fixes)
   - No clear verdict → message the reviewer asking for a clear verdict
   - Maximum reached (5 rounds) → proceed to Step 6 with a note

### Step 4: Apply fixes

Based on the reviewer's findings, the **lead** (you) fixes the issues:

**For plan review:** update the plan — address each finding.

**For code review:** edit files, run tests if applicable.

Show the user:
```
### Fixes (Round N)
- [What was changed and why, one item per finding]
```

**Skip** a fix if it contradicts the user's explicit requirements — note
this for the user.

### Step 5: Request re-review (Rounds 2–5)

Before sending, check that the teammate is still reachable:
- Verify that SendMessage is available as a tool
- If the tool is missing or the call returns an error — the teammate
  is no longer active, skip to the fallback below

If the teammate is reachable, send a message with the list of fixes:

```
I've revised based on your feedback.

Here's what I changed:
[List of fixes from Step 4]

Re-review with the same adversarial stance. Focus on:
1. Whether my fixes actually resolve the reported issues
2. Any NEW issues introduced by the fixes

End with VERDICT: APPROVED or VERDICT: REVISE.
```

If the reviewer responds — return to **Step 3**.

**If the reviewer does not respond** (teammate is no longer active):

1. **Operator available** (interactive session — the conversation was
   initiated by a human message, not a CI trigger or scheduled run) — ask:
   ```
   The reviewer is no longer active. Fixes have been applied:
   [List of fixes from Step 4]

   Options:
   (a) Spawn a new reviewer to verify fixes (expensive — full project re-read)
   (b) Conclude the review — fixes applied, verification is on you
   ```
   If the operator chooses (a) — spawn a new reviewer with this briefing:
   ```
   You are reviewing code changes in this repository.

   This is a re-review (Round N). A previous reviewer found issues
   that have been addressed.

   ## Previous findings
   [Verbatim findings from Round N-1]

   ## Fixes applied
   [List of fixes from Step 4]

   Verify whether fixes resolve the findings. Check for new issues.

   End with VERDICT: APPROVED or VERDICT: REVISE.
   ```
   Continue from Step 3.
   If the operator chooses (b) — proceed to Step 6.

2. **Operator not available** (headless, CI, scheduled run) — proceed
   to Step 6 with status "fixes applied, not re-verified":
   ```
   ## Team Review — Summary (mode: <mode>)

   **Status:** Fixes applied, re-verification not completed
   (reviewer became inactive after Round N)

   **Applied fixes:**
   [List of fixes per finding]

   **Note:** First-round findings were addressed but not re-verified
   by the reviewer. Manual review of fixes is recommended.
   ```

### Step 6: Final result

**Approved:**
```
## Team Review — Summary (mode: <mode>)

**Status:** Approved after N round(s)

[Final review]

---
**Reviewed and approved by the reviewer teammate. Awaiting your decision.**
```

**Maximum rounds reached:**
```
## Team Review — Summary (mode: <mode>)

**Status:** Maximum reached (5 rounds) — not fully approved

**Remaining findings:**
[Unresolved issues]

---
**The reviewer still has findings. Please review them and decide how to proceed.**
```

### Step 7: Cleanup

If the Agent Teams runtime provides a team cleanup mechanism, use it.
Failures are non-blocking — teammates are cleaned up when the session ends.

Do NOT delete plan files that existed before the review.

---

## Rules

- Lead **actively fixes** issues — this is NOT just message forwarding
- Reviewer findings shown **verbatim** — do not rephrase or shorten
- Auto-detect mode from context; user arguments take priority
- The reviewer **never writes files** — enforced by agent definition
- The reviewer **can run commands** (tests, linters, git) and **use MCP**
  (Context7, web search) to verify findings
- Maximum 5 rounds to protect against infinite loops
- Show the user reviews and fixes for each round
- If Agent Teams are not enabled — tell the user how to enable them
- Avoid creating auxiliary files (memory files, state files, logs, temporary
  markdown) — prefer working within the conversation context
- If a fix contradicts user requirements — skip and explain why
- For re-review rounds, try to continue the existing reviewer teammate
  first. If the teammate is no longer active, decide by context: ask the
  operator when available, or conclude without re-verification in headless
  mode. Re-spawning a new reviewer is expensive (full project re-read) —
  offer it as an option, not as the default.
- The ultimate goal is **higher quality** of plans, code, and other
  artifacts. Token economy is a means, not an end — never skip a
  verification step or cut a round short just to save tokens.

---

## Comparison with adversarial-review

| Aspect                 | adversarial-review (Codex)     | claude-team-review (Teams)     |
|------------------------|--------------------------------|--------------------------------|
| Reviewer model         | External (GPT via Codex CLI)   | Claude (same model family)     |
| Cross-model blind spots| Yes — different model biases   | No — same model, different context |
| Session persistence    | Via `codex exec resume`        | Try to continue teammate; graceful fallback if inactive |
| External dependencies  | Codex CLI + OpenAI API key     | None — built into Claude Code  |
| Reviewer capabilities  | Read-only sandbox              | Read + execute + MCP + web     |
| Context isolation       | Full (different model)         | Full (separate context window) |
| Token cost per round   | External API (OpenAI pricing)  | Claude tokens (Max plan)       |
