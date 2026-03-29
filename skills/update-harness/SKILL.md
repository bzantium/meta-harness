---
name: update-harness
description: "Update an existing Claude Code harness — check for outdated components, duplication with official features, missing patterns, and improvement opportunities, then apply approved changes. Use whenever: 'update harness', 'update my harness', 'is my harness up to date', 'harness outdated', 'optimize harness', 'check my harness', 'review harness', 'harness health check', 'refresh harness'. Triggers on: 'update', 'harness update', 'harness check', 'what should I update', 'are my agents still needed'."
---

# update-harness — Harness Health Check & Updater

Analyzes an existing Claude Code harness against current official features, project evolution, and the pattern library. Produces actionable recommendations: what to remove, update, and add.

## Workflow

### Phase 1: Inventory

Scan the existing harness and catalog every component:

1. **Agents** — Read all `.claude/agents/*.md`: name, model, role, constraints
2. **Skills** — Read all `.claude/skills/*/SKILL.md`: name, triggers, workflow
3. **Hooks** — Read `.claude/settings.json` hooks section: events, matchers, scripts
4. **Rules** — Read `CLAUDE.md`: delegation rules, conventions, agent catalog
5. **MCP** — Read `.mcp.json` if present
6. **Plugins** — Read `.claude/settings.json` enabled plugins

Produce an inventory list with component count. If no harness is found, inform the user and suggest using `create-harness` instead.

### Phase 2: Official Features Delta

Check what the harness duplicates or could leverage from official features:

1. Read `../create-harness/references/official-builtins.md`
2. WebSearch "Claude Code changelog latest features" for anything newer than the reference
3. For each harness component, evaluate:
   - **Redundant?** — Does an official feature now do the same thing? (e.g., a custom explore agent when built-in Explore exists)
   - **Superseded?** — Has a new official feature made this component unnecessary? (e.g., custom task tracking when TaskCreate is now built-in)
   - **Leverageable?** — Could an official feature enhance this component? (e.g., a skill could use CronCreate instead of custom scheduling)

### Phase 3: Project Evolution

Spawn **scout** to re-analyze the current project state:

```
Agent(
  name: "scout",
  prompt: "Analyze this project. Follow your investigation protocol and produce the structured analysis report.",
  model: "sonnet"
)
```

Compare the scout's current findings against what the harness covers:

| Change | Implication |
|---|---|
| New modules/services added | Harness may need new agents or wider scope |
| Modules removed | Harness may reference dead code |
| Tech stack changed (new framework, build tool) | Hooks/skills may target wrong tools |
| Test framework changed | Verification hooks may be broken |
| Team size changed (solo to team or vice versa) | Harness scale may be wrong |
| New external integrations (APIs, databases) | May need safety hooks or specialized agents |

### Phase 4: Pattern Gap Analysis + Deferred Pattern Check

1. Read `../create-harness/references/pattern-library.md`
2. **Check Deferred Patterns** — If the harness was created by meta-harness, look for a "Deferred Patterns" table in the harness summary or CLAUDE.md. For each deferred pattern, check if its trigger condition is now met (e.g., "Add Verification loop when first test file is created" → do tests now exist?)
3. Read `../create-harness/references/pattern-library.md` and apply its decision matrix against the **current** project state

4. Identify patterns that:
   - **Missing:** Apply to the current project but aren't in the harness
   - **Obsolete:** Are in the harness but no longer fit (project simplified or changed direction)
   - **Upgradeable:** Have better implementations available in the pattern library
   - **Deferred → Ready:** Trigger condition from deferred table is now met

### Phase 5: Component Quality Review

**Model Capability Re-assessment:** Every harness component encodes an assumption about what the model can't do on its own. As models improve, those assumptions go stale. For each component, ask: "Does the current model handle this well enough without this component?" If yes, recommend removal. Reference: newer models (e.g., Opus 4.6) handle long-context, self-correction, and planning better — components that compensated for older model weaknesses may now be dead weight.

**Self-evaluation Bias Check:** Agents evaluating their own work tend to be overly generous. Flag any harness component where the same agent both generates AND evaluates output. Recommend separating generator from evaluator when found.

Review each existing component for quality issues:

- **Agents:** Model assignments still optimal? (reference `../create-harness/references/templates.md` Model Assignment Guide). Descriptions specific enough to trigger?
- **Skills:** Descriptions pushy enough? No trigger overlaps? Workflow matches current project?
- **Hooks:** Event types correct? Scripts functional with current tooling?
- **CLAUDE.md:** Agent catalog matches actual files? Build/test commands runnable?

**Gotcha Review:** Read `.forge/gotchas.md` if it exists. This file is populated by the Gotcha Capture hook across sessions. For each HIGH confidence gotcha:
- Map it to a specific harness component (agent, skill, hook, or rule)
- Add a concrete UPDATE or ADD recommendation to the report
For MEDIUM confidence gotchas:
- Present to the user for confirmation before including in recommendations

### Phase 5B: Verification

Before presenting the report, spawn a **critic** subagent to review the recommendations:

```
Agent(
  prompt: "Review these harness update recommendations for quality.

PROJECT ANALYSIS:
{scout output from Phase 3}

PROPOSED RECOMMENDATIONS:
{all REMOVE/UPDATE/ADD/KEEP items from Phases 2-5}

Check for:
1. FALSE REMOVALS — Is something marked for removal that's still load-bearing?
2. FALSE ADDITIONS — Is something recommended to add that the project doesn't actually need yet?
3. SELF-EVALUATION BIAS — Any component where the same agent generates and evaluates?
4. MODEL CAPABILITY — Are recommendations accounting for current model capabilities, or based on older model assumptions?
5. DEFERRED PATTERN TRIGGERS — Were deferred patterns checked? Are any triggers met but missed?
6. SCALE — After applying all changes, would the harness still match the project's scale tier?
7. GOTCHA COVERAGE — Were all HIGH confidence gotchas from .forge/gotchas.md addressed? Any gotcha that maps to a harness fix should appear in the recommendations.

Output: APPROVE or REVISE with specific corrections.",
  model: "sonnet"
)
```

If REVISE, apply corrections before presenting the report.

### Phase 6: Report

Present all findings as a single actionable report:

```
## Harness Audit Report

**Project:** {name}
**Components:** {N} agents, {N} skills, {N} hooks
**Last modified:** {from file dates or git history}

### REMOVE — Duplicates official features
- {component}: {reason} — Use built-in {feature} instead

### UPDATE — Improvement opportunities
- {component}: {what to change and why}

### ADD — New recommendations
- {pattern/component}: {why it's now relevant, what changed}

### KEEP — Still valuable
- {component}: {why it's still needed}

### Health Summary
{2-3 sentences: overall assessment, most critical action item, estimated effort}
```

Rules for the report:
- Every recommendation must have a **reason** — never just "add X"
- REMOVE items must name the specific built-in replacement
- ADD items must reference which pattern or feature they're based on
- KEEP items confirm what's working — don't skip this section
- If the harness is healthy, say so clearly. Do not invent problems.

**After presenting the report, use `AskUserQuestion` to ask the user which recommendations to apply.** Do not auto-apply any changes. Proceed to Phase 7 only after explicit approval.

### Phase 7: Apply

Apply only the items the user approved. Work by category:

**REMOVE:**
1. Delete agent/skill files that are no longer needed
2. Remove corresponding hooks from `.claude/settings.json`
3. Delete hook scripts (`.claude/hooks/`) if no other hook references them
4. Remove entries from CLAUDE.md agent catalog and delegation rules

**UPDATE:**
1. Edit agent frontmatter (model changes, description improvements)
2. Rewrite skill descriptions for better trigger matching
3. Update hook scripts or event types
4. Sync CLAUDE.md agent catalog with actual agent files
5. Update build/test commands in CLAUDE.md if they changed

**ADD:**
1. Generate new agent files using templates from `../create-harness/references/templates.md`
2. Generate new skill files with pushy descriptions (follow Skill Description Writing Rules in templates)
3. Add new hooks to `.claude/settings.json` and create hook scripts in `.claude/hooks/`
4. Update CLAUDE.md: add new agents to catalog, add delegation rules, add conventions

After all changes, run a mini-validation:
- Agent names in CLAUDE.md match actual agent files
- Hooks in settings.json reference scripts that exist
- No skill trigger overlaps introduced
- Component count still matches project scale

Present a summary of what was changed.

## Constraints

- Phases 1-6 are read-only — do NOT modify any files until Phase 7
- Phase 7 requires explicit user approval for each category (REMOVE/UPDATE/ADD)
- Do NOT recommend components that duplicate official features
- Do NOT recommend increasing scale beyond what the project needs
- If there's nothing to report in a section, write "None" — do not omit the section
