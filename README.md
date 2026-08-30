# j-QA

An agent skill for **manual, human-perspective QA through the real running UI** — not a code review. It scopes itself to the current git changes, walks the product as a domain expert would, and ends in a **SHIP / DON'T-SHIP** verdict with a defect log.

Compatible with **Claude Code** and **Cursor** (both use a `SKILL.md` skill folder). Needs a **running app** and browser tools.

## What this skill does

A feature is not done because the diff looks right and tests are green. It is done when a real person can find it, use it, trust the number it produces, come back later and still see that number, and prove it in the backend.

The skill:

- Derives **expected behaviour from the changed code** (the oracle), not from memory
- Designs persona-based scenarios (ISTQB techniques, Nielsen heuristics, UAT discipline)
- Executes them in the **browser** with evidence
- Names the **verification rung** reached: UI-live > DOM > API > DB > code-oracle
- Adapts phases to the change (and writes why each phase was included or skipped)
- Calls an impact sweep for neighbouring-code regression instead of re-implementing it
- Delivers a **SHIP / DON'T-SHIP** verdict plus a prioritised defect log

`j-QA` **uses the running app**. Sibling skills `feature-impact-sweep` and `adversarial-review` **read the code**. `j-user-readiness` asks whether a stranger can understand the screen.

## When to use it

Invoke when you say `j-QA`, `QA this`, `manually test the UI`, `test it like a real user`, or after building any user-facing feature.

## Install

### Cursor

**User-level:**

```bash
git clone https://github.com/Jayant2907/j-QA.git ~/.cursor/skills/j-QA
```

Windows (PowerShell):

```powershell
git clone https://github.com/Jayant2907/j-QA.git "$env:USERPROFILE\.cursor\skills\j-QA"
```

**Project-level:**

```bash
git clone https://github.com/Jayant2907/j-QA.git .cursor/skills/j-QA
```

### Claude Code

**User-level:**

```bash
git clone https://github.com/Jayant2907/j-QA.git ~/.claude/skills/j-QA
```

Windows (PowerShell):

```powershell
git clone https://github.com/Jayant2907/j-QA.git "$env:USERPROFILE\.claude\skills\j-QA"
```

**Project-level:**

```bash
git clone https://github.com/Jayant2907/j-QA.git .claude/skills/j-QA
```

Alternatively, copy the folder (it must contain `SKILL.md`) into the same locations.

Restart the agent session so the skill is picked up.

## How to run it

1. Have the app running locally (or at a reachable URL).
2. Have the change in git (the skill scopes to current changes).
3. Ask: `Run j-QA on this change` or `QA this like a real user`.

Fill in the skill's "Notes to adapt per codebase" (app URL, how to log in, where the audit trail lives) so the agent can actually drive your product.

## What's in this repo

```
j-QA/
├── SKILL.md    # Full QA procedure, rungs, verdict contract
└── README.md
```
