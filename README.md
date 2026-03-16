# Plan Reviewer

An AI skill that reviews implementation plans before you start coding. Point it at a plan, and it'll read your actual codebase to find what the plan missed — gaps, edge cases, broken assumptions — then fix the plan and hand it back.

Works with any AI coding agent (Cursor, Copilot, Claude Code, Windsurf, etc).

## What happens when you run it

The skill walks through your plan in phases:

1. **Reads and summarizes** the plan to confirm it understands what you're building
2. **Hunts for gaps** by checking the plan against real code — missing files, broken import chains, forgotten error handling
3. **Thinks about edge cases** — concurrency, null values, partial failures, retry behavior
4. **Checks UI concerns** — loading states, error messages, responsiveness, accessibility
5. **Dry-runs the plan** against the codebase — traces execution paths, follows data flows, simulates failures
6. **Updates the plan** with fixes for everything it found
7. **Asks questions** it couldn't answer from the code alone

## Pick your depth

Not every plan needs the full treatment.

| Depth | What runs | Good for |
|---|---|---|
| `quick` | Gaps + light dry-run | Small changes, config, low-risk stuff |
| `standard` | Gaps + edge cases + dry-run | Most plans (this is the default) |
| `deep` | Everything, including UI review | Big features, risky refactors, frontend work |

Say "quick review", "review the plan", or "deep review" to trigger it.

## Install

Drop the folder into your agent's skill directory:

```bash
# Using the skills.sh CLI (auto-detects your agent)
npx skills add <your-username>/plan-reviewer

# Or manually copy to your agent's skill folder, e.g.:
# ~/.agents/skills/plan-reviewer/
# .cursor/skills/plan-reviewer/
```

## What's in the box

```
plan-reviewer/
├── SKILL.md              # The actual skill instructions
├── README.md
└── examples/
    ├── before.md         # A sample plan (intentionally incomplete)
    └── after.md          # The same plan after a standard review
```

Check out the `examples/` folder to see what a reviewed plan looks like.
