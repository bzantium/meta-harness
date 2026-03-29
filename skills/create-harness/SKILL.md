---
name: create-harness
description: "Generate a complete, project-specific Claude Code harness — agents, skills, hooks, CLAUDE.md rules — tailored to any codebase. Use whenever someone says 'create a harness', 'set up agents for this project', 'configure Claude Code', 'build an agent team', 'make this project AI-ready', 'harness this project', or any request to create Claude Code agent infrastructure. Also triggers on: 'harness', 'agent setup', 'skill setup', 'hook configuration', 'generate agents'. Automatically checks official built-in features to prevent duplication."
---

# meta-harness — Intelligent Harness Creator

Generates project-specific Claude Code harnesses by analyzing the codebase, selecting proven patterns from a curated library, and producing agents, skills, hooks, and rules — while auto-checking official features to avoid duplication.

## Workflow

### Phase 0: Deduplication Pre-check

Before any design work, establish what already exists:

1. **Read the built-in features reference** — Load `references/official-builtins.md` to know what Claude Code provides natively (50+ commands, 25 hook events, 5 subagent types, Agent Teams, etc.)

2. **Live update check** — Use WebSearch to query "Claude Code changelog latest features" to catch anything newer than the reference. If new built-in features are found that overlap with what you'd generate, note them.

3. **Check target project** — Look for existing `.claude/` directory contents:
   - `.claude/agents/` — existing agent definitions
   - `.claude/skills/` — existing skill definitions
   - `.claude/settings.json` — enabled plugins, hook configs
   - `CLAUDE.md` — existing project rules
   - `.mcp.json` — MCP server configs

4. **Compile exclusion list** — Everything found above is off-limits for generation. Present the list to the user.

If the user requests something that's already built-in, inform them and suggest the native feature instead.

### Phase 0.5: Clarification (if needed)

Before analysis, evaluate whether the user's request provides enough context. Use `AskUserQuestion` to gather missing information if:
- The project is empty/new and the user didn't describe what they're building
- The user's description is vague (e.g., "harness this project" with no context)
- Key decisions affect harness design (team size, deployment target, timeline)

Example questions to ask (use the system popup, NOT text):
- "What is the main purpose of this project?" (with options based on what you can infer)
- "What's the team size?" (Solo / Small 2-5 / Medium 6-15 / Large 15+)
- "What's the primary workflow?" (Feature development / Bug fixes / Research / Content creation)

**Skip this phase entirely if:**
- The project has substantial code (scout can infer everything)
- The user provided a detailed description with their request (e.g., `/create-harness FastAPI backend with PostgreSQL, 3-person team`)

### Phase 1: Project Analysis

Spawn the **scout** agent to analyze the target project:

```
Agent(
  name: "scout",
  prompt: "Analyze this project at {project_path}. Follow your investigation protocol and produce the structured analysis report.",
  model: "sonnet"
)
```

Review the scout's output. If the project is empty or has no code AND Phase 0.5 was skipped, use `AskUserQuestion` to ask what kind of project they're building before proceeding.

### Phase 2: Pattern Selection

Spawn a **pattern-selector** subagent to select patterns. This MUST be a subagent — do NOT select patterns in the main conversation, as conversation history will bias the selection.

```
Agent(
  prompt: "Select optimal patterns for this project.

PROJECT ANALYSIS:
{scout output from Phase 1}

DECISION MATRIX — match project traits to patterns:

| Project Trait | Recommended Patterns |
|---|---|
| Any project | Three-tier model routing, Progressive disclosure |
| Modifies files | Destructive command guard (safety hook) |
| Has tests | Verification loop |
| Multi-module | Fan-out parallel agents |
| API + Frontend | Three-layer separation (plan/execute/review) |
| Security-sensitive | READ-ONLY advisor enforcement, Security reviewer agent |
| Long autonomous tasks | Persistence loop (Ralph pattern) |
| Team collaboration | Proactive delegation |
| AI-generated code risk | Anti-slop cleanup |
| Cross-session work | Boulder state persistence |
| Protected directories | Edit freeze zones |
| Repeated workflows | Continuous learning |
| Quality-critical output (code, content, design) | Generator-Evaluator pattern |
| High-level spec → concrete implementation | Sprint contracts |
| Sessions > 1-2 hours | Context reset |

Read `references/pattern-library.md` for full pattern details.

RULES:
- Base your selection ONLY on the project analysis above
- Do NOT consider any topics or context outside the project analysis
- For each selected pattern, cite the specific project trait from the analysis that triggered it
- For each EXCLUDED pattern, provide a concrete reason tied to the project analysis — never use vague reasons like 'unnecessary' or 'overkill'
- Your selections must be defensible. If someone questions an exclusion, the reason you gave should already answer their question

Output format:

## Selected Patterns
| Pattern | Triggered By | Reason |
|---|---|---|
| {name} | {specific project trait from analysis} | {why this pattern fits} |

## Excluded Patterns
| Pattern | Reason for Exclusion |
|---|---|
| {name} | {concrete reason tied to project analysis — e.g., 'No test framework detected in analysis', not 'not needed'} |

## Scale Estimate
- Selected patterns: {N}
- Estimated agents: {N} (map patterns to agent roles — multiple patterns can share one agent)
- Estimated skills: {N}
- Estimated hooks: {N}
- Scale tier: {lightweight | standard | full | extended}",
  model: "sonnet"
)
```

Then spawn a **pattern-critic** subagent to review the selections before presenting to the user:

```
Agent(
  prompt: "Review this pattern selection for quality and consistency.

PROJECT ANALYSIS:
{scout output from Phase 1}

PROPOSED SELECTION:
{pattern-selector output}

Review each decision against the project analysis and check for:

1. FALSE POSITIVES — Selected patterns that the project doesn't actually need
   - Is there a concrete project trait supporting this, or is it assumed?
   - Would this pattern add complexity without proportional value at this project's scale?

2. FALSE NEGATIVES — Excluded patterns that should have been selected
   - Does the exclusion reason hold up? Check it against the actual project analysis
   - Is a pattern excluded as 'overkill' when a lightweight version would fit?

3. REASONING QUALITY — Are exclusion reasons specific and defensible?
   - Flag any vague reasons ('not needed', 'overkill', 'unnecessary')
   - Each exclusion must cite a concrete absence in the project analysis

4. FOCUS AREA CROSS-CHECK — Compare excluded patterns against the scout's Recommended Focus Areas
   - If a focus area implies a pattern that was excluded, flag as potential false negative
   - Example: scout says "credential security is a focus" but Security reviewer is excluded → flag

5. SCALE CHECK — Compare the Scale Estimate against the architect's scale guidelines
   - Solo small: 2-3 agents, 1-2 skills, 0-1 hooks
   - Solo medium: 3-5 agents, 2-3 skills, 1-2 hooks
   - Team medium: 4-6 agents, 3-5 skills, 2-3 hooks
   - Team large: 5-8 agents, 4-6 skills, 3-5 hooks
   - If estimated agents exceed the guideline, flag which patterns to merge or defer

6. CURRENT vs PLANNED — Check if any pattern was selected based on Planned State rather than Current State
   - Patterns triggered by planned-but-not-yet-existing traits should note this explicitly
   - For empty/greenfield projects: only select patterns for what exists NOW, defer the rest with triggers

Output format:

## Verdict
{APPROVE | REVISE}

## Issues Found (if REVISE)
| Pattern | Issue | Recommendation |
|---|---|---|
| {name} | {false positive/negative/vague reasoning} | {what to change} |

## Final Selection (if REVISE, provide corrected list)
| Pattern | Triggered By | Reason |
|---|---|---|
| {name} | {trait} | {reason} |

## Final Exclusions (if REVISE, provide corrected list)
| Pattern | Reason for Exclusion |
|---|---|
| {name} | {concrete reason} |",
  model: "sonnet"
)
```

If the critic returns REVISE, apply the corrections. If APPROVE, proceed.

Then **present the final selections to the user**:
- Show both selected AND excluded patterns with their full reasoning
- The excluded patterns table is as important as the selected one — users need to see WHY each was excluded

After presenting the summary, use `AskUserQuestion` to get approval. The user may want to add or remove patterns.

**When the user questions a selection or exclusion:**
- Do NOT immediately reverse the decision
- Re-examine the scout's project analysis for evidence
- If the evidence supports the original decision, explain why and stand by it
- Only change if the user provides NEW information not in the scout analysis (e.g., "we're adding tests next week", "this will be a team project soon")
- If changing, state what new information justified the change

### Phase 3: Architecture Design

Spawn the **architect** agent with the collected inputs:

```
Agent(
  name: "architect",
  prompt: "Design a harness for this project.

PROJECT ANALYSIS:
{scout output from Phase 1}

SELECTED PATTERNS:
{pattern selections from Phase 2}

DEDUP CONSTRAINTS (do NOT duplicate these):
{exclusion list from Phase 0}

Follow your design process and produce the harness specification.",
  model: "opus"
)
```

**Review the architecture** before proceeding:
- Every agent has an explicit model assignment
- No component duplicates a built-in feature
- Scale matches the project (not over-engineered)
- Skill descriptions are specific enough to trigger correctly

Present the architecture summary to the user, then use `AskUserQuestion` to get approval before generating files.

### Phase 4: File Generation

Generate all harness files based on the approved architecture. Read `references/templates.md` for file templates.

#### 4.1 Agent Definitions

For each agent in the architecture, create `{project}/.claude/agents/{name}.md`:

- YAML frontmatter: name, description, model
- Sections: Core Role, Work Principles, Tool Usage, Constraints
- READ-ONLY agents: explicitly list Write/Edit in disallowed or constrained tools
- Include specific, actionable instructions (not vague role descriptions)

#### 4.2 Skill Definitions

For each skill, create `{project}/.claude/skills/{name}/SKILL.md`:

- YAML frontmatter: name, description (write it "pushy" — aggressive trigger matching)
- Body under 500 lines
- Include: when to trigger, step-by-step workflow, output expectations

**References directory:** Create `{project}/.claude/skills/{name}/references/` when the skill needs supporting content that would bloat SKILL.md beyond 300 lines. Common reference types:
- **Domain knowledge** — API specs, schema definitions, protocol docs the skill needs to consult
- **Decision tables** — detailed criteria, scoring rubrics, or evaluation matrices
- **Templates** — output format templates, file scaffolds, boilerplate the skill generates
- **Examples** — sample inputs/outputs, worked examples for complex workflows
- **Checklists** — validation criteria, review checklists, QA procedures

SKILL.md references these with inline pointers: "For details on X, read `references/x.md`"
The skill body contains the workflow; references contain the knowledge.

#### 4.3 Hook Configuration

Add hooks to `{project}/.claude/settings.json`:

- Only generate hooks with clear, measurable value
- Safety hooks (PreToolUse): shell scripts that check commands
- Quality hooks (PostToolUse): formatting, validation
- **Gotcha Capture hook (Stop)**: Always include. Analyzes the session for harness-level issues and appends to `.forge/gotchas.md`. See `references/templates.md` for the template.
- Include the actual hook scripts if needed

#### 4.4 Project Rules (CLAUDE.md)

Generate `{project}/CLAUDE.md` with:

- Agent catalog table (name, role, model, when to use)
- Model routing rules
- Project-specific conventions
- Delegation rules (when to spawn which agent)
- Deferred Patterns table (from Phase 2 excluded patterns with trigger conditions)
- Do NOT include generic coding advice Claude already knows

#### 4.5 Generation Exclusions

Do NOT generate:
- README, CHANGELOG, or documentation about the harness itself
- Installation guides or setup instructions as separate files
- Meta-information about the generation process
- `.forge/` directory (created at runtime, not at generation time — but add it to the directory structure docs in CLAUDE.md)

### Phase 5: Validation

After generation, verify the harness:

1. **Structure** — All files in correct paths, valid YAML frontmatter
2. **Dedup** — Cross-check every component against Phase 0 exclusion list
3. **Consistency** — Agent names in skills match actual agent files
4. **Triggers** — For each skill, verify with test queries:
   - **Should trigger:** 3+ phrases that must activate the skill
   - **Should NOT trigger:** 2+ phrases that must NOT activate it
   - Fix descriptions that fail either test. For details → `references/skill-testing-guide.md`
5. **Models** — Every agent has a model assignment
6. **Scale** — Component count matches project complexity
7. **Depth** — No delegation chain deeper than 2 levels

Fix any issues found. Then summarize what was generated:

```
## Harness Summary

Generated {N} agents, {N} skills, {N} hooks, CLAUDE.md

### Agents
- {name} ({model}) — {role}

### Skills
- {name} — {trigger description}

### Hooks
- {event}: {purpose}

### Usage
Ready to use. Agents, skills, and hooks are now active.

### Deferred Patterns
| Pattern | Add When... |
|---|---|
| {pattern} | {concrete trigger condition} |
```

The **Deferred Patterns** table is required for every harness. For each excluded pattern, if it could become relevant as the project evolves, include the specific trigger condition. This gives the user a roadmap for harness evolution and makes `update-harness` more effective by providing predefined checkpoints.

## Anti-duplication Rules

These are NEVER generated because they're built-in:

- File search → built-in Glob tool
- Content search → built-in Grep tool
- Code exploration → built-in Explore subagent
- Plan creation → built-in /plan command + Plan subagent
- Task tracking → built-in TaskCreate/TaskUpdate tools
- Context management → built-in /compact, /context, /rewind
- Git operations → built-in Bash tool
- MCP connections → built-in /mcp command
- Agent teams → built-in TeamCreate/SendMessage (experimental)
- Code intelligence → built-in LSP tool

If a user requests any of these, point them to the native feature.

## References

- Pattern details → `references/pattern-library.md`
- Built-in features → `references/official-builtins.md`
- File templates → `references/templates.md`
- Skill testing → `references/skill-testing-guide.md`
- Team examples → `references/team-examples.md`
- QA patterns → `references/qa-patterns.md`

## Test Scenarios

### Normal: Solo web developer
1. User: "Create a harness for this Next.js project"
2. Dedup: no conflicts, no existing harness
3. Scout: Next.js + TypeScript, solo dev, medium complexity
4. Patterns: model routing, verification loop, destructive guard
5. Architect: 3 agents, 2 skills, 1 hook
6. Generate and validate

### Normal: Team API project
1. User: "Set up agents for our Python API"
2. Dedup: existing CLAUDE.md found → preserve, extend
3. Scout: FastAPI + PostgreSQL, 5 contributors, large
4. Patterns: three-layer separation, proactive delegation, security reviewer
5. Architect: 6 agents, 4 skills, 2 hooks
6. Generate and validate

### Conflict: Duplicate request
1. User: "Create a code search agent"
2. Dedup: Explore subagent is built-in
3. Response: "Claude Code already has a built-in Explore agent for code search. No need to generate one."

### Conflict: Existing harness
1. User: "Create a harness" (project already has agents/)
2. Dedup: finds 4 existing agents, 2 skills
3. Response: "Found existing harness with 4 agents and 2 skills. Want to extend it or redesign?"
