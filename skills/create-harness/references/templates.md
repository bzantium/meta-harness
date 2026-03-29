# Generation Templates

Use these templates during Phase 4 (File Generation) to produce consistent, high-quality harness files.

---

## Agent Definition Template

File: `{project}/.claude/agents/{name}.md`

```markdown
---
name: {agent-name}
description: "{One-line description of what this agent does. Be specific about when to use it.}"
model: {opus|sonnet|haiku}
---

# {Agent Name} — {Role Title}

{One paragraph explaining the agent's core responsibility and why it exists as a separate agent.}

## Core Role

{2-3 sentences defining the agent's primary function and boundaries.}

## Work Principles

1. {Principle 1 — what guides this agent's decisions}
2. {Principle 2}
3. {Principle 3}

## Tool Usage

- **Primary tools:** {list the tools this agent uses most}
- **Restricted:** {tools this agent should NOT use, and why}

## Output Expectations

{What the agent produces: file changes, reports, analysis documents, etc.}

## Constraints

- {Hard constraint 1}
- {Hard constraint 2}
- {When to escalate or stop}
```

### Agent Naming Conventions

- Use lowercase kebab-case: `security-reviewer`, not `SecurityReviewer`
- Name by role, not technology: `api-designer`, not `openapi-writer`
- Keep names short (1-2 words): `planner`, `executor`, `reviewer`

### Model Assignment Guide

| Agent Role | Model | Reasoning |
|---|---|---|
| Codebase search, file lookup | haiku | Speed over depth |
| Code implementation, test writing | sonnet | Balanced quality/cost |
| Planning, architecture review | opus | Deep reasoning needed |
| Security analysis | sonnet | Pattern matching, not invention |
| Code review (logic) | opus | Catches subtle bugs |
| Documentation (API docs, JSDoc) | haiku | Mechanical signature-to-description |
| Documentation (architecture, design) | sonnet | Requires contextual understanding |

---

## Skill Definition Template

File: `{project}/.claude/skills/{name}/SKILL.md`

```markdown
---
name: {skill-name}
description: "{Pushy description. List ALL things this skill does. List ALL trigger phrases. End with 'Use this skill whenever...' or 'Triggers on:'. Be aggressive about when to activate.}"
---

# {Skill Name}

{One-line summary of what this skill accomplishes.}

## When to Use

{Bullet list of specific situations that trigger this skill.}

## Workflow

### Step 1: {Action}
{Instructions}

### Step 2: {Action}
{Instructions}

### Step N: {Final Action}
{Instructions}

## Output

{What the skill produces when complete.}
```

### Skill References Directory

When a skill needs domain knowledge, decision tables, templates, or examples, create a `references/` subdirectory alongside SKILL.md:

```
.claude/skills/{name}/
├── SKILL.md                # Workflow (under 500 lines)
└── references/
    ├── {domain}.md          # Domain knowledge the skill consults
    ├── {criteria}.md        # Decision tables, scoring rubrics
    ├── {templates}.md       # Output format templates, scaffolds
    └── {examples}.md        # Worked examples, sample inputs/outputs
```

**When to create references:**
- SKILL.md body exceeds ~300 lines → split heavy content into references
- The skill needs to consult static knowledge (API specs, schemas, protocols)
- The skill has complex decision logic (evaluation matrices, multi-factor scoring)
- The skill generates structured output from templates

**When NOT to create references:**
- Skill is simple enough to fit in SKILL.md (<200 lines)
- Reference content is already available via WebSearch or project docs
- Content would be a single paragraph (just inline it)

**How SKILL.md references them:**
```markdown
### Step 3: Evaluate
Apply the scoring criteria from `references/evaluation-rubric.md` to grade each output.
```

### Skill Description Writing Rules

The description is the ONLY trigger mechanism. Claude is conservative about triggering, so write descriptions aggressively:

**Bad:** `"Runs tests for the project"`
**Good:** `"Run all project tests — unit, integration, e2e. Execute test suites, check coverage, report failures. Use whenever: 'run tests', 'check tests', 'test this', 'verify it works', 'does it pass', or after any code change that should be validated."`

**Bad:** `"Reviews code quality"`
**Good:** `"Multi-aspect code review: logic bugs, security vulnerabilities, performance issues, maintainability, test coverage gaps. Triggers on: 'review this', 'check my code', 'is this good', 'code review', 'PR review', or proactively after significant code changes."`

Include:
1. Everything the skill does (not just the primary action)
2. Every trigger phrase a user might say
3. Proactive triggers (when it should auto-activate)

### Orchestrator Skill Template

Generate only when 4+ agents need coordinated workflows:

```markdown
---
name: {domain}-orchestrator
description: "Coordinate the {domain} agent team. Manages execution order, data flow between agents, and error handling. Triggers on: '{domain} workflow', 'run the full pipeline', 'coordinate agents', 'orchestrate', or any multi-agent task request."
---

# {Domain} Orchestrator

Coordinates the agent team for {domain} workflows.

## Agents

| Agent | Role | Model |
|---|---|---|
| {name} | {role} | {model} |

## Workflow

### Step 1: {Phase name}
Spawn **{agent}** with: {input description}
Expected output: {what the agent produces}

### Step 2: {Phase name}
Spawn **{agent}** with: {input from step 1}
Expected output: {what the agent produces}

{Continue for each phase...}

## Error Handling

- If {agent} fails at Step N: {retry strategy or fallback}
- If output quality is insufficient: {retry with more context or escalate}
- Maximum retry per step: 2

## Data Flow

{agent A} → {artifact} → {agent B} → {artifact} → {agent C}
Intermediate artifacts stored in `.forge/` at runtime.
```

---

## Hook Configuration Templates

Hooks go in `{project}/.claude/settings.json` under the `hooks` key.

### Destructive Command Guard

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/check-careful.sh"
          }
        ]
      }
    ]
  }
}
```

Script (`.claude/hooks/check-careful.sh`):
```bash
#!/bin/bash
# Read tool input from stdin
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | python3 -c "import sys,json; print(json.load(sys.stdin).get('input',{}).get('command',''))" 2>/dev/null)

# Dangerous patterns (customize per project)
DANGEROUS_PATTERNS=(
  "rm -rf /"
  "rm -rf ~"
  "git push.*--force.*main"
  "git push.*--force.*master"
  "DROP TABLE"
  "DROP DATABASE"
  "DELETE FROM"
  "docker system prune -a"
  "> /dev/sd"
  "mkfs\\."
  "dd if="
)

# Safe exceptions
SAFE_PATTERNS=(
  "rm -rf node_modules"
  "rm -rf dist"
  "rm -rf build"
  "rm -rf .next"
  "rm -rf __pycache__"
  "rm -rf .pytest_cache"
  "rm -rf target"
  "DELETE FROM.*WHERE"
)

for safe in "${SAFE_PATTERNS[@]}"; do
  if echo "$COMMAND" | grep -qiE "$safe"; then
    echo '{}'
    exit 0
  fi
done

for pattern in "${DANGEROUS_PATTERNS[@]}"; do
  if echo "$COMMAND" | grep -qiE "$pattern"; then
    echo "{\"permissionDecision\": \"ask\", \"message\": \"WARNING: Potentially destructive command detected: $COMMAND\"}"
    exit 0
  fi
done

echo '{}'
```

### Auto-Format After Edit

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/auto-format.sh"
          }
        ]
      }
    ]
  }
}
```

Script runs the project's formatter (prettier, black, gofmt, etc.) on the edited file.

### Session Summary on Stop

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Before ending, save a brief summary of what was accomplished this session to .claude/session-log.md (append, don't overwrite). Include: what was done, what's left, any decisions made."
          }
        ]
      }
    ]
  }
}
```

### Gotcha Capture (Stop) — Always Include

This hook runs at session end to capture harness-level issues for `update-harness` to consume later.

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Analyze this session for HARNESS-LEVEL gotchas only. Ignore normal coding bugs or one-time mistakes.\n\nLook for these signals:\n1. USER CORRECTIONS of agent behavior (e.g., 'don't modify files', 'use sonnet not opus for this')\n2. REPEATED 'don't do X' instructions (same correction given 2+ times)\n3. HOOKS blocking legitimate operations (false positives in safety guards)\n4. SKILLS not triggering when they should (user had to invoke manually)\n5. MISSING RULES that the user had to state explicitly (e.g., 'always run tests after changes')\n\nFor each gotcha found, classify confidence:\n- HIGH: Clear harness deficiency (user explicitly said the harness should do X differently)\n- MEDIUM: Likely harness issue (repeated friction pattern)\n- LOW: Might be one-time issue (skip these)\n\nAppend only HIGH and MEDIUM gotchas to .forge/gotchas.md in this format:\n\n## {date}\n- [{HIGH|MEDIUM}] {category}: {description} → {suggested harness fix}\n\nIf no harness-level gotchas were found in this session, do NOT write anything. Do not invent problems."
          }
        ]
      }
    ]
  }
}
```

The `.forge/gotchas.md` file accumulates across sessions. `update-harness` reads it during Phase 5 to generate data-driven improvement recommendations.

---

## CLAUDE.md Template

File: `{project}/CLAUDE.md`

```markdown
# {Project Name}

{One-line project description}

## Agent Catalog

| Agent | Model | Role | When to Use |
|---|---|---|---|
| {name} | {model} | {role} | {trigger condition} |

## Delegation Rules

- For code implementation: use {implementer agent}
- For code review: use {reviewer agent}
- For codebase exploration: use built-in Explore agent
- For planning: use built-in Plan mode (/plan)
{project-specific delegation rules}

## Conventions

{Only include conventions that differ from framework defaults}
- {Convention 1}
- {Convention 2}

## Testing

- Framework: {test framework}
- Run: `{test command}`
- Coverage minimum: {percentage or "none"}

## Build

- `{build command}`
- `{dev command}`
- `{lint command}`
```

### CLAUDE.md Writing Rules

1. **No generic advice** — Don't include "write clean code" or "use meaningful names." Claude knows this.
2. **Only project-specific info** — Conventions, commands, agent catalog, delegation rules.
3. **Commands must be runnable** — Every command listed should work in the project.
4. **Keep it under 100 lines** — CLAUDE.md is loaded into every conversation. Shorter = less context waste.

---

## Directory Structure for Generated Harness

```
{project}/
├── .claude/
│   ├── agents/
│   │   ├── {agent-1}.md
│   │   ├── {agent-2}.md
│   │   └── ...
│   ├── skills/
│   │   └── {skill-name}/
│   │       ├── SKILL.md
│   │       └── references/     (if skill body exceeds 300 lines)
│   │           └── {detail}.md
│   ├── hooks/
│   │   ├── check-careful.sh    (if safety hook selected)
│   │   └── auto-format.sh      (if format hook selected)
│   └── settings.json           (updated with hook configs)
├── CLAUDE.md                   (project rules)
└── .forge/                 (created at runtime, NOT at generation time)
    └── {phase}_{agent}_{artifact}.{ext}
```

### .forge/ Convention

- Created at runtime when agents produce intermediate artifacts, NOT during harness generation
- Naming: `{phase}_{agent}_{artifact}.{ext}` (e.g., `01_planner_architecture.md`, `02_implementer_api-routes.ts`)
- Purpose: agent handoff artifacts, audit trail, debugging
- Add `.forge/` to `.gitignore` — these are ephemeral, not version-controlled
- CLAUDE.md should mention: "Intermediate artifacts are stored in `.forge/`. Do not commit this directory."
