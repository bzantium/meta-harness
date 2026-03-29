# Pattern Library

Curated from 4 production-proven harness projects. Each pattern includes: what it is, when to use it, and how to implement it in a Claude Code harness.

---

## 1. Agent Delegation Patterns

### 1.1 Proactive Delegation

Agents auto-dispatch based on context rather than explicit user request. When a code change is made, the reviewer triggers automatically. When security-sensitive code is touched, the security agent activates.

**When to use:** Team projects where quality gates should be automatic, not opt-in.

**Implementation:**
- In CLAUDE.md, define delegation rules: "When {condition}, spawn {agent}"
- Example rules:
  - Complex request → spawn planner agent first
  - Code changes in auth/ → spawn security-reviewer
  - Test files modified → spawn test-engineer to verify coverage
- Agents are spawned via Agent tool with `run_in_background: true` for non-blocking review

### 1.2 Three-Layer Separation

Separates work into Planning → Execution → Review layers with different specialists at each level. Planners can only write plans (READ-ONLY to code). Executors implement. Reviewers validate.

**When to use:** Complex projects where planning quality directly impacts implementation success.

**Implementation:**
- Planning agents: opus model, READ-ONLY (no Write/Edit tools)
- Execution agents: sonnet model, full tool access
- Review agents: opus model, READ-ONLY for code review; full access for test execution
- Handoff via files in `.forge/` or via TaskCreate

### 1.3 Role-Based Specialists

Each agent is a specialist with a specific role in a development sprint: product diagnostic → scope review → architecture lockdown → implementation → QA → deployment.

**When to use:** Full-lifecycle projects where each phase needs distinct expertise.

**Implementation:**
- Define agents by development phase, not by technology
- Each agent's output becomes the next agent's input
- Skills orchestrate the pipeline: plan → implement → review → ship

---

## 2. Model Routing Patterns

### 2.1 Three-Tier Routing

Assign models by cognitive demand:
- **haiku** — Fast search, simple lookups, documentation ($)
- **sonnet** — Code implementation, testing, standard work ($$)
- **opus** — Planning, architecture, complex reasoning ($$$)

**When to use:** Always. This is the default pattern for any harness.

**Implementation:**
- In agent definition frontmatter: `model: {haiku|sonnet|opus}`
- Agent tool calls: `model: "{tier}"`
- Decision heuristic: if the agent needs to make judgment calls → opus. If it executes a known workflow → sonnet. If it just searches → haiku.

### 2.2 Category-Based Routing

Instead of per-agent model assignment, tasks are routed by semantic category: `visual-engineering`, `deep-reasoning`, `quick-lookup`, `code-generation`. Each category maps to an optimal model.

**When to use:** Multi-model environments where the same agent might use different models for different subtasks.

**Implementation:**
- Define categories in CLAUDE.md with model mappings
- Agents check task category before spawning sub-work
- Most harnesses should use 2.1 (simpler) unless they need this flexibility

---

## 3. Safety Patterns

### 3.1 Destructive Command Guard

A PreToolUse hook on Bash that intercepts commands and blocks destructive operations (rm -rf, DROP TABLE, force-push, etc.) with a warning, while allowing safe variants (rm -rf node_modules/).

**When to use:** Any project that modifies files. This is a near-universal pattern.

**Implementation:**
Hook in settings.json:
```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "bash .claude/hooks/check-careful.sh"
      }]
    }]
  }
}
```

Script reads JSON from stdin, extracts command, pattern-matches against dangerous patterns:
- `rm -rf /` (but not `rm -rf node_modules/`)
- `git push --force` to main/master
- `DROP TABLE`, `DELETE FROM` without WHERE
- `docker system prune`

Returns `{"permissionDecision": "ask", "message": "Warning: ..."}` to block, or `{}` to allow.

### 3.2 Edit Freeze Zones

PreToolUse hooks on Edit and Write that deny file operations outside a specified directory boundary. Useful for protecting production configs, generated files, or vendor code.

**When to use:** Projects with directories that should never be modified by AI (e.g., vendor/, generated/, .env files).

**Implementation:**
- Store freeze boundary in a config file (e.g., `.claude/freeze-dir.txt`)
- Hook script resolves target file path and checks if it's within the allowed boundary
- Returns `{"permissionDecision": "deny", "message": "..."}` for violations

### 3.3 READ-ONLY Agent Enforcement

Advisor agents (architects, critics, reviewers) are structurally prevented from modifying code. This is done via a PreToolUse hook that checks the current agent's identity and blocks Write/Edit tools for designated READ-ONLY agents.

**When to use:** Projects where separation of concerns between advisors and implementers matters.

**Implementation:**
- In agent frontmatter, mark with a `readonly: true` flag or note in description
- PreToolUse hook checks agent identity (from hook context) against a READ-ONLY list
- Blocks Write/Edit/Bash(write operations) for those agents

---

## 4. Persistence Patterns

### 4.1 Ralph Loop (Persistence Until Done)

A Stop hook re-injects work instructions when the agent stops, forcing it to continue until verification passes or max iterations are reached. Named after the Sisyphean metaphor — the boulder never stops rolling.

**When to use:** Long-running autonomous tasks where stopping early means wasted work. Not for interactive sessions.

**Implementation:**
- Stop hook checks if there are incomplete todos/tasks
- If incomplete: re-inject a prompt like "You have {N} incomplete tasks. Continue working."
- Track iteration count to enforce maximum (typically 5-10)
- Include a kill switch (e.g., user says "stop" or "cancel")

### 4.2 Boulder State Persistence

Save work state to a JSON file so multi-session projects can resume where they left off. The "boulder" file tracks: active plan, completed tasks, current phase, accumulated learnings.

**When to use:** Projects that span multiple Claude Code sessions (multi-day work).

**Implementation:**
- State file: `{project}/.claude/boulder.json`
- Schema: `{ plan: string, phase: string, completedTasks: string[], learnings: string[] }`
- SessionStart hook loads boulder state and injects context
- Task completion updates the boulder file
- A `/start-work` skill resumes from boulder state

---

## 5. Quality Patterns

### 5.1 Verification Loop

A 6-phase quality gate: Build → Types → Lint → Tests → Security → Diff Review. Can run periodically (every 15 minutes) during long sessions or as a pre-commit check.

**When to use:** Any project with a build/test pipeline. Scale the phases to what the project actually uses.

**Implementation:**
- Create a skill that runs applicable phases in sequence
- Skip phases the project doesn't have (no TypeScript → skip Types)
- On failure: report which phase failed and what needs fixing
- Can be triggered manually or via a CronCreate for periodic runs

### 5.2 Anti-Slop Cleanup

A dedicated pass to clean AI-generated code bloat: unnecessary comments, over-abstracted helpers, redundant error handling, verbose type annotations. Uses a deletion-first mindset — if removing code doesn't break tests, it shouldn't be there.

**When to use:** After any significant AI-generated code session, especially for user-facing code.

**Implementation:**
- Create a skill with a writer/reviewer separation
- Step 1: Lock existing behavior with tests (before any deletion)
- Step 2: Review agent identifies bloat candidates
- Step 3: Delete candidates one by one, running tests after each
- Step 4: If tests pass, the deletion stands

### 5.3 Progressive QA

QA runs incrementally after each module completion, not just once at the end. The QA agent performs "cross-boundary comparison" — reading both sides of an interface simultaneously.

**When to use:** Multi-module projects where integration bugs are likely.

**Implementation:**
- QA agent uses `general-purpose` type (needs to run scripts)
- After each module: QA reads the module's API + its consumer code
- Checks: type compatibility, error handling consistency, data shape matching
- Reports mismatches before they compound

---

## 6. Context Management Patterns

### 6.1 Progressive Disclosure

Three tiers of context loading to manage token budget:
- **Tier 1 (always loaded):** Skill name + description (~100 words)
- **Tier 2 (on trigger):** SKILL.md body (<500 lines)
- **Tier 3 (on demand):** references/ subdirectory (unlimited)

**When to use:** Always. Every skill should follow this structure.

**Implementation:**
- Skill description: concise but complete trigger information
- SKILL.md body: workflow steps, decision tables, quick reference
- references/: detailed guides, templates, examples, domain-specific content
- In SKILL.md, use pointers: "For details on X, read `references/x.md`"

### 6.2 Wisdom Accumulation

After each subagent completes a task, extract learnings into categories (Conventions, Successes, Failures, Gotchas) and pass them to ALL subsequent subagents. Task 5 knows what Task 1 discovered.

**When to use:** Multi-task orchestrations where later tasks benefit from earlier discoveries.

**Implementation:**
- After each agent task, append findings to a shared notepad file
- Categories: `Conventions`, `Successes`, `Failures`, `Gotchas`, `Commands`
- Next agent's prompt includes: "Previous learnings: {notepad content}"
- Store in `.forge/learnings.md` for audit trail

---

## 7. Learning Patterns

### 7.1 Continuous Learning

A Stop hook evaluates completed sessions (minimum 10 messages), detects patterns (error resolutions, user corrections, workarounds), and saves them as reusable knowledge.

**When to use:** Projects with repeated workflows where session-to-session improvement matters.

**Implementation:**
- Stop hook runs an evaluation script
- Script analyzes conversation for: repeated errors, user corrections, successful approaches
- Saves patterns to `~/.claude/learned/{project}/` as markdown files
- SessionStart hook loads relevant learned patterns back into context
- Include a confidence score to eventually prune low-value learnings

### 7.2 Gotcha Capture

A Stop hook that analyzes each session for harness-level issues — agent behavior corrections, hook false positives, missing rules, skill trigger failures — and appends them to `.forge/gotchas.md` with confidence ratings. `update-harness` reads this file to generate data-driven improvement recommendations.

**When to use:** Always. Every generated harness should include this hook. It's the feedback loop that makes `update-harness` effective.

**Implementation:**
- Stop hook with prompt type analyzes session for 5 signal categories:
  1. User corrections of agent behavior
  2. Repeated "don't do X" instructions (2+ times)
  3. Hooks blocking legitimate operations
  4. Skills not triggering when expected
  5. Missing rules the user had to state explicitly
- Each gotcha classified as HIGH/MEDIUM/LOW confidence
- Only HIGH and MEDIUM appended to `.forge/gotchas.md`
- Format: `- [{confidence}] {category}: {description} → {suggested fix}`
- `.forge/gotchas.md` is gitignored (ephemeral, project-local)

---

## 8. Orchestration Patterns

### 8.1 Pipeline
Sequential dependent tasks: A → B → C. Each agent's output feeds the next.
**Best for:** Phased workflows (plan → implement → test → review).

### 8.2 Fan-out/Fan-in
Parallel independent tasks merged at the end. Multiple agents work simultaneously on different files/modules.
**Best for:** Multi-file changes, bulk operations, independent analyses.

### 8.3 Expert Pool
A router examines each task and selects the most appropriate specialist.
**Best for:** Help-desk style workflows where request types vary widely.

### 8.4 Producer-Reviewer
Generate then validate with a retry loop (max 2-3 iterations).
**Best for:** Code generation + review, content creation + editing.

### 8.5 Supervisor
A central agent monitors work, redistributes tasks dynamically, and resolves conflicts.
**Best for:** Large projects with many interdependent tasks.

---

## 9. Harness Architecture Patterns

### 9.1 Generator-Evaluator (GAN-inspired)

A separate evaluator agent grades the generator's output against explicit criteria, providing concrete feedback for iteration. The generator and evaluator are NEVER the same agent — self-evaluation bias causes agents to praise their own work even when quality is mediocre.

**When to use:** Any workflow where output quality is critical and verifiable — code generation, content creation, design. Especially valuable when quality criteria can be codified (e.g., "does this pass tests?", "does this meet the design spec?").

**Implementation:**
- Generator agent produces output (code, content, design)
- Evaluator agent reviews against explicit criteria with scoring rubric
- If any criterion falls below threshold → feedback to generator for another round
- Max 2-3 iterations to prevent infinite loops
- Evaluator should use tools to VERIFY, not just read (e.g., run tests, navigate UI with Playwright)
- Calibrate the evaluator with few-shot examples of good/bad grading — uncalibrated evaluators are too lenient

### 9.2 Sprint Contracts

Before work begins, generator and evaluator negotiate a contract: what "done" looks like for each chunk of work. Bridges the gap between high-level specs and testable implementation.

**When to use:** When a planner produces high-level specs but the generator needs concrete acceptance criteria. Prevents the generator from building the wrong thing and the evaluator from testing the wrong criteria.

**Implementation:**
- Generator proposes: what it will build + how success will be verified
- Evaluator reviews the proposal and pushes back on vague or untestable criteria
- Both iterate until they agree on the contract
- Generator builds against the contract
- Evaluator grades against the contract (not the original spec)
- Communication via files: one agent writes, another reads and responds

### 9.3 Harness Simplification

Every harness component encodes an assumption about what the model can't do on its own. Those assumptions go stale as models improve. Periodically strip away components that are no longer load-bearing.

**When to use:** After model upgrades, or when a harness feels sluggish/expensive. This is the core principle behind `update-harness`.

**Implementation:**
- For each component, ask: "Does the current model handle this well enough without this component?"
- Remove one component at a time and evaluate impact (don't strip everything at once)
- Components that compensated for older model weaknesses (context anxiety, poor self-correction) are prime candidates
- The space of interesting harness combinations doesn't shrink as models improve — it moves

### 9.4 Context Reset

For long-running sessions, clear the context window entirely and start a fresh agent with a structured handoff artifact, rather than relying on compaction alone. Compaction preserves continuity but doesn't eliminate context anxiety or coherence decay.

**When to use:** When sessions exceed 1-2 hours or when the model starts losing coherence, wrapping up prematurely, or repeating itself.

**Implementation:**
- At natural breakpoints (sprint completion, phase transition), create a handoff artifact:
  - What was accomplished
  - Current state of the work
  - Next steps
  - Key decisions made and constraints discovered
- Spawn a fresh agent with ONLY the handoff artifact as context
- The fresh agent gets a clean slate — no accumulated confusion
- Note: Opus 4.6 largely handles long context natively, so test whether resets are still needed for your model

---

## Pattern Composition

Patterns combine well. Common compositions:

| Composition | Use Case |
|---|---|
| Three-tier routing + Destructive guard | Minimum viable harness for any project |
| Pipeline + Producer-reviewer | Plan → Implement → Review with retry |
| Fan-out + Wisdom accumulation | Parallel work that shares discoveries |
| Ralph loop + Verification loop | Autonomous work with quality gates |
| Proactive delegation + Anti-slop | Auto-trigger review + cleanup |
| Three-layer separation + Progressive QA | Enterprise-grade development pipeline |

### Anti-slop + Wisdom Accumulation + Continuous Learning

These three patterns overlap in the "learning/quality" space. When using all three together, enforce strict responsibility separation:

| Pattern | Scope | When | What it produces |
|---|---|---|---|
| Anti-slop cleanup | Single agent's output | Immediately after code generation | Cleaned code (deletions) |
| Wisdom accumulation | Between agents in one session | After each agent completes a task | `.forge/learnings.md` entries for next agent |
| Continuous learning | Between sessions | On session Stop | `~/.claude/learned/{project}/` pattern files |

**Do NOT combine these into one agent or one hook.** Each has a distinct trigger timing (post-generation vs post-task vs post-session) and a distinct output target (code vs shared notepad vs persistent files).
