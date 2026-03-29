# meta-harness

**The harness that builds harnesses**

You don't configure a harness by hand. You say _"harness this project"_ and meta-harness analyzes your codebase, selects from 20 battle-tested patterns, and generates a complete set of agents, skills, hooks, and rules — tailored to your project, deduplicated against Claude Code's built-in features.

## Why meta-harness?

Existing harness projects are powerful but general-purpose. They ship 28+ agents and 125+ skills designed for _everyone_. meta-harness takes the opposite approach: it studies **your** project and generates only what **you** need.

- A solo Next.js app gets 3 agents, 2 skills, 1 hook
- A team API backend gets 5 agents, 3 skills, 2 hooks
- Neither gets bloat the other doesn't need

## What It Generates

```
{your-project}/
├── .claude/
│   ├── agents/           # Domain-specific agents with model assignments
│   ├── skills/           # Project-specific workflow skills
│   ├── hooks/            # Safety guards + quality automation scripts
│   └── settings.json     # Hook configurations
├── CLAUDE.md             # Agent catalog, delegation rules, conventions
└── .forge/           # Runtime artifacts (created at use time)
```

## How It Works

### create-harness — Build a new harness

```
Phase 0  Dedup Pre-check      Official builtins + WebSearch + existing components
Phase 1  Project Analysis      Scout (sonnet) explores your codebase
Phase 2  Pattern Selection     Decision matrix maps project traits → patterns
Phase 3  Architecture Design   Architect (opus) designs the harness
Phase 4  File Generation       Agents, skills, hooks, CLAUDE.md
Phase 5  Validation            7-point check + deferred patterns table for harness evolution
```

### update-harness — Keep it current

Projects evolve. Claude Code adds features. Harnesses go stale. `update-harness` catches up:

```
Phase 1  Inventory             Catalog existing harness components
Phase 2  Official Delta        Find builtins that now supersede your components
Phase 3  Project Evolution     Re-analyze for new modules, changed stack, team growth
Phase 4  Pattern Gap           Identify missing or obsolete patterns
Phase 5  Quality Review        Model assignments, trigger quality, hook validity
Phase 6  Report                REMOVE / UPDATE / ADD / KEEP with reasons
Phase 7  Apply                 Execute approved changes + mini-validation
```

## Installation

**Via Marketplace**
```
/plugin marketplace add bzantium/meta-harness
/plugin install meta-harness@meta-harness
```

**From local clone**
```
git clone https://github.com/bzantium/meta-harness.git
/plugin install meta-harness
```

**As Global Skill**
```
cp -r skills/* ~/.claude/skills/
```

## Usage

```
/create-harness
/update-harness
```

If another skill has the same name, use the fully qualified form:
```
/meta-harness:create-harness
/meta-harness:update-harness
```

You can pass a project description as an argument:
```
/create-harness FastAPI backend with PostgreSQL and Redis, 3-person team
/update-harness check if any agents are now redundant
```

Or just say it naturally:

- _"Create a harness for this project"_
- _"Set up agents for this codebase"_
- _"Is my harness up to date?"_
- _"Update my harness"_

## Before / After

**Without a harness:**
```
"Review this code"          → opus for a trivial lint check
"Help me deploy"            → no safety guard, accidentally rm -rf
Next session                → starts from zero, no context carried over
```

**With meta-harness:**
```
Lint check                  → routed to haiku (fast, cheap)
Deploy command              → destructive guard intercepts dangerous ops
Next session                → boulder state + gotcha capture carry context forward
```

## Examples

### Simple — Personal Blog (Hugo, solo)

```
/create-harness
```

```
.claude/
├── agents/
│   ├── implementer.md        (sonnet — content writing, template editing)
│   └── reviewer.md           (opus — catches broken layouts, READ-ONLY)
├── skills/
│   └── publish/              (hugo build → frontmatter check → git push)
├── hooks/
│   └── check-careful.sh
└── settings.json
```

2 agents, 1 skill, 1 hook. Deferred: verification loop (add when tests exist), edit freeze (add when vendored themes appear).

---

### Intermediate — E-commerce API (Django + Stripe, 3-person team)

```
/create-harness Django REST API with Stripe payments, team of 3
```

```
.claude/
├── agents/
│   ├── planner.md            (opus — API design, migration planning, READ-ONLY)
│   ├── implementer.md        (sonnet — code + tests)
│   ├── reviewer.md           (opus — logic review, cross-boundary checks, READ-ONLY)
│   ├── security-reviewer.md  (sonnet — auth flows, payment code)
│   └── test-engineer.md      (sonnet — coverage gaps, fixtures)
├── skills/
│   ├── dev-pipeline/         (plan → implement → review → security)
│   ├── verify/               (black → ruff → pytest → diff)
│   ├── cleanup/              (anti-slop: lock tests → find bloat → delete → verify)
│   └── api-orchestrator/     (5-agent coordination for cross-module features)
├── hooks/
│   ├── guard-destructive.sh  (DROP TABLE, DELETE, force-push)
│   └── auto-format.sh        (black + isort on save)
└── settings.json
```

5 agents, 4 skills, 3 hooks. Auto-triggers: security-reviewer spawns on `payments/` or `auth/` changes. Critic caught sprint contracts as premature for a mature codebase — deferred.

---

### Advanced — Trading Platform (Rust + Python + TypeScript, 8-person team)

```
/create-harness Low-latency trading system, 8 engineers, strict correctness requirements
```

```
.claude/
├── agents/
│   ├── planner.md            (opus — cross-service architecture, READ-ONLY)
│   ├── implementer.md        (sonnet — Rust/Python/TypeScript)
│   ├── security-reviewer.md  (opus — financial data, PII, compliance, READ-ONLY)
│   ├── reviewer.md           (opus — logic correctness, READ-ONLY)
│   ├── qa-engineer.md        (sonnet — integration tests, k6 latency benchmarks)
│   └── proto-guardian.md     (sonnet — protobuf contract compatibility, READ-ONLY)
├── skills/
│   ├── dev-orchestrator/     (fan-out across services, fan-in for integration)
│   ├── verification-suite/   (cargo test → pytest → vitest → lint → k6 → diff)
│   ├── sprint-contract/      (planner-reviewer contract negotiation)
│   ├── anti-slop/            (changed files only, deletion-first)
│   └── resume-work/          (boulder state resumption)
├── hooks/
│   ├── check-careful.sh      (rm -rf, DROP, force-push, kubectl delete)
│   ├── freeze-zones.sh       (proto/*.proto, terraform/, CI workflows)
│   └── auto-format.sh        (rustfmt, black, prettier)
└── settings.json
```

6 agents, 5 skills, 5 hooks. Critic merged language-specific implementers into one, added proto-guardian for contract safety. Proactive delegation: proto/ changes auto-trigger proto-guardian, auth/ changes auto-trigger security-reviewer.

## Pattern Library

20+ patterns organized by concern:

| Category | Patterns |
|----------|----------|
| **Delegation** | Proactive dispatch, Three-layer separation, Role-based specialists |
| **Model Routing** | Three-tier routing (haiku/sonnet/opus), Category-based routing |
| **Safety** | Destructive command guard, Edit freeze zones, READ-ONLY enforcement |
| **Persistence** | Ralph loop, Boulder state, Context reset |
| **Quality** | Verification loop, Anti-slop cleanup, Progressive QA |
| **Context** | Progressive disclosure, Wisdom accumulation |
| **Learning** | Continuous learning, Gotcha capture |
| **Architecture** | Generator-Evaluator, Sprint contracts, Harness simplification |
| **Orchestration** | Pipeline, Fan-out/Fan-in, Expert pool, Producer-Reviewer, Supervisor |

## Design Decisions

**Three-tier model routing** — Not every agent needs opus. Search agents use haiku, implementation uses sonnet, only planning and review use opus. This is the default for every generated harness.

**Scout + Architect separation** — Analysis and design are different skills. Scout (sonnet) collects data fast. Architect (opus) reasons about it deeply. Neither modifies files.

**4-axis agent separation** — Before adding an agent, it must justify itself on at least 2 of: Expertise, Parallelism, Context isolation, Reusability. This prevents agent sprawl.

**Official dedup** — A built-in features reference is checked before every generation. WebSearch catches features added after the reference was written. If Claude Code already does it, we don't generate it.

**2-level delegation limit** — Agent A can delegate to Agent B, but B should not delegate to C. Deeper chains lose context and compound errors.

## Plugin Structure

```
meta-harness/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── agents/
│   ├── scout.md                        # Project analyzer (sonnet)
│   └── architect.md                    # Harness designer (opus)
├── skills/
│   ├── create-harness/
│   │   ├── SKILL.md                    # Creation workflow
│   │   └── references/
│   │       ├── pattern-library.md      # 16 patterns from 5 projects
│   │       ├── official-builtins.md    # Claude Code built-in feature registry
│   │       ├── templates.md            # Generation templates
│   │       ├── skill-testing-guide.md  # Trigger validation methodology
│   │       ├── team-examples.md        # 3 reference configurations
│   │       └── qa-patterns.md          # Bug detection patterns
│   └── update-harness/
│       └── SKILL.md                    # Update workflow
└── CLAUDE.md
```

## Requirements

- Claude Code v2.1+
- Agent Teams (optional): `export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`

## Acknowledgments

The pattern library is curated from real-world architectures of these projects:

- [Everything Claude Code](https://github.com/affaan-m/everything-claude-code)
- [gstack](https://github.com/garrytan/gstack)
- [oh-my-claudecode](https://github.com/Yeachan-Heo/oh-my-claudecode)
- [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent)

## License

MIT
