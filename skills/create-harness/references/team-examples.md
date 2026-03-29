# Team Configuration Examples

Three reference configurations for common project types. Use as starting points — adapt agent count, models, and roles to the specific project.

---

## Example 1: Web Application (Next.js + Database)

**Scale:** Standard (solo/small team)
**Execution Mode:** subagent
**Patterns:** Three-tier routing, Destructive command guard, Verification loop

| Agent | Model | Role | Key Constraint |
|---|---|---|---|
| explorer | haiku | Codebase search, file discovery | READ-ONLY |
| implementer | sonnet | Feature implementation, bug fixes | Must run tests after changes |
| reviewer | opus | Code review, architecture decisions | READ-ONLY |

**Skills:**
- `dev-workflow`: Implement → lint → test → review pipeline
- `db-migrate`: Schema change workflow with backup reminder

**Hooks:**
- PreToolUse/Bash: Destructive command guard (rm -rf, DROP TABLE)

**Why this works:** Solo/small team doesn't need specialists. Three generalists cover the full workflow. Haiku for speed on search, sonnet for the bulk of implementation, opus only for judgment calls.

---

## Example 2: API Backend (Python FastAPI + PostgreSQL, 5-person team)

**Scale:** Full
**Execution Mode:** subagent
**Patterns:** Three-layer separation, Proactive delegation, Destructive command guard, Progressive QA

| Agent | Model | Role | Key Constraint |
|---|---|---|---|
| explorer | haiku | Codebase + API endpoint discovery | READ-ONLY |
| planner | opus | Architecture, API design, migration planning | READ-ONLY |
| implementer | sonnet | Code implementation, test writing | Must follow planner's spec |
| api-reviewer | sonnet | Endpoint consistency, schema validation | READ-ONLY |
| security-reviewer | opus | Auth flows, SQL injection, data exposure | READ-ONLY |

**Skills:**
- `api-workflow`: Plan → implement → review → security check pipeline
- `migration-workflow`: Schema change with rollback plan
- `endpoint-review`: Cross-check endpoint definitions against OpenAPI spec

**Hooks:**
- PreToolUse/Bash: Destructive guard (DROP, DELETE, force-push)
- PreToolUse/Edit: Freeze zone on `alembic/versions/` (no manual migration edits)

**Why this works:** Team project needs separation between planning and implementation. Security reviewer at opus because auth bugs are subtle. API reviewer at sonnet because schema checks are pattern-based.

---

## Example 3: CLI Tool (Go, solo developer)

**Scale:** Lightweight
**Execution Mode:** subagent
**Patterns:** Three-tier routing, Verification loop

| Agent | Model | Role | Key Constraint |
|---|---|---|---|
| implementer | sonnet | Feature implementation, test writing | Must run `go test` after changes |
| reviewer | opus | Design review, edge case analysis | READ-ONLY |

**Skills:**
- `release-workflow`: Test → build → tag → changelog update

**Hooks:**
- PostToolUse/Edit: Auto-format with `gofmt`

**Why this works:** CLI tools are typically single-binary, single-language. Two agents are enough. No explorer needed — Go's LSP handles navigation. Opus reviewer catches edge cases in flag parsing, error handling.

---

## Anti-Patterns to Avoid

| Anti-Pattern | Why It's Bad | Instead |
|---|---|---|
| 1 agent per file type | Agents should map to roles, not files | Group by responsibility |
| Reviewer that also implements | Separation of concerns violated | Split into two agents |
| Opus for everything | Cost/speed waste on simple tasks | Three-tier routing |
| 8 agents for a solo project | Orchestration overhead exceeds value | 2-3 agents max |
| Skill that wraps a single tool | No value over using the tool directly | Delete the skill |
