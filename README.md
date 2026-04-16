# Claude Team Review

Adversarial code and plan review using Claude Code Agent Teams.

One teammate reviews. The lead fixes. Iterate until approved.

## What is this

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code) that
spawns an adversarial reviewer as an Agent Teams teammate. The reviewer
reads your project, runs tests, checks documentation, and delivers findings
with a skeptical stance. The lead (your main session) fixes issues and
requests re-review from the same teammate. If the teammate is no longer
active, the lead decides how to proceed — re-spawn or conclude.

### How it differs from [adversarial-review](https://github.com/dementev-dev/adversarial-review)

**adversarial-review** uses two different models (Claude writes, Codex
reviews) — you get cross-model blind spot coverage and cheap re-review
via `codex exec resume`. It requires Codex CLI and an OpenAI API key.

**claude-team-review** stays within the Claude ecosystem. No external
dependencies. The reviewer is a Claude Code teammate with its own context
window, MCP access, and the ability to run commands. For re-review, the
lead tries to continue the same teammate; if the teammate is no longer
active, the lead can re-spawn or conclude based on context. The trade-off:
same model family means no cross-model diversity.

Use **adversarial-review** when you want maximum review quality through
model diversity. Use **claude-team-review** when you want zero external
dependencies and a richer reviewer (tests, docs, web search).

## How it works

```
┌──────────┐     spawn        ┌────────────┐
│   Lead   │ ───────────────> │  Reviewer   │
│  (code)  │                  │ (teammate)  │
└──────────┘                  └────────────┘
     ^                              │
     │          findings            │
     │ <────────────────────────────┘
     │
     │  fix issues
     v
┌──────────┐     message      ┌────────────┐
│   Lead   │ ───────────────> │  Reviewer   │
│  (fixed) │  "re-check this" │ (same / new)│
└──────────┘                  └────────────┘
                                    │
                              VERDICT: APPROVED
```

The lead tries to continue the **same teammate** for re-review. If the
teammate is no longer active (Agent Teams limitation), the lead can
re-spawn with a full briefing or conclude without re-verification.

### Three modes

| Mode           | What it reviews                    | When to use              |
|----------------|------------------------------------|--------------------------|
| `plan`         | Implementation plan                | Before writing code      |
| `code`         | Git diff (unstaged, staged, branch)| After writing code       |
| `code-vs-plan` | Code changes against the plan      | Verify implementation    |

Mode is auto-detected from context, or you can force it with an argument.

### What the reviewer can do

- **Read** any file in the repository
- **Run commands** — tests, linters, type checkers, build scripts
- **Search the web** and **query documentation** via MCP (Context7)
- **Inspect git history** — blame, log, diff

The reviewer **cannot** create, edit, or delete project files.

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) ≥ 2.1.32
- Agent Teams enabled (experimental)

No external dependencies. No API keys beyond your Claude subscription.

## Installation

```bash
# Clone the repository
git clone https://github.com/dementev-dev/claude-team-review.git
cd claude-team-review

# Symlink the skill
ln -s "$(pwd)" ~/.agents/skills/claude-team-review

# Symlink the reviewer agent definition
mkdir -p ~/.claude/agents
ln -s "$(pwd)/adversarial-reviewer.md" ~/.claude/agents/adversarial-reviewer.md
```

Enable Agent Teams in your Claude Code settings:

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

Restart Claude Code after installation for the skill to be recognized.

## Usage

```bash
# Auto-detect what to review
/claude-team-review

# Review a plan
/claude-team-review plan

# Review code changes
/claude-team-review code

# Review a specific file
/claude-team-review path/to/plan.md

# Use maximum reasoning effort for the reviewer
/claude-team-review xhigh
```

## Reviewer behavior

The reviewer uses an adversarial stance — it defaults to skepticism
and tries to break confidence in the change. Each finding must answer:

1. **What can go wrong?** — concrete scenario
2. **Why vulnerable?** — cite specific location
3. **Impact** — what breaks and how badly
4. **Recommendation** — specific fix

The reviewer verifies findings by running tests, checking documentation,
and inspecting related code before reporting.

## Roadmap

- [ ] Real-world testing and iteration on prompts
- [ ] Parallel multi-reviewer mode (security + performance + correctness)
- [ ] Persistent reviewer memory across sessions
- [ ] Integration with CI (GitHub Actions)
- [ ] Comparison benchmarks: Codex backend vs Team backend

## Эксперимент: сравнение ревьюеров

Мы запустили оба ревьюера (Opus и GPT-5.4) на одном и том же плане
и сравнили находки. Ключевой вывод: модели ревьюят из принципиально
разных парадигм — Opus как архитектор ("сработает ли этот дизайн?"),
Codex как security/ops инженер ("что сломается в продакшене?").
Ноль полных совпадений, ~30% частичных пересечений.

Подробности: [EXPERIMENT.md](EXPERIMENT.md) — полный ход эксперимента,
все находки, анализ пересечений, выводы.

## Related

- [adversarial-review](https://github.com/dementev-dev/adversarial-review) —
  cross-model variant using Codex CLI as the reviewer backend
- [Claude Code Agent Teams docs](https://code.claude.com/docs/en/agent-teams) —
  official documentation on Agent Teams

## License

Apache-2.0 — see [LICENSE](LICENSE).
