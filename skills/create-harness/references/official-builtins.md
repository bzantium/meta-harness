# Official Claude Code Built-in Features

**Last updated:** 2026-03-29 (Claude Code v2.1.x)

Use this reference during Phase 0 (Deduplication Pre-check) to avoid generating components that duplicate native functionality. Also run a live WebSearch for "Claude Code changelog" to catch anything newer.

---

## Built-in Slash Commands (50+)

Do NOT create skills that replicate these:

| Command | Purpose |
|---|---|
| /plan | Enter plan mode (read-only exploration) |
| /compact | Compact conversation with optional focus |
| /context | Visualize context usage |
| /diff | Interactive diff viewer for uncommitted changes |
| /doctor | Diagnose installation and settings |
| /clear, /reset | Clear conversation history |
| /resume, /continue | Resume a previous session |
| /rewind, /checkpoint | Rewind conversation and code state |
| /cost | Show token usage statistics |
| /model | Select or change AI model |
| /fast | Toggle fast mode |
| /effort | Set effort level (low/medium/high/max/auto) |
| /mcp | Manage MCP server connections |
| /plugins | Manage plugins |
| /permissions | View/update tool permissions |
| /hooks | View hook configurations |
| /agents | Manage agent configurations |
| /skills | List available skills |
| /tasks | List/manage background tasks |
| /schedule | Create/manage cloud scheduled tasks |
| /security-review | Analyze branch changes for security |
| /pr-comments | Fetch GitHub PR comments |
| /export | Export conversation as text |
| /voice | Toggle voice dictation |
| /vim | Toggle Vim mode |
| /sandbox | Toggle sandbox mode |
| /statusline | Configure status line |
| /stats | Visualize daily usage |
| /release-notes | View changelog |
| /init | Initialize project with CLAUDE.md |
| /memory | Edit CLAUDE.md memory files |

**Also built-in as skills:**
| Skill | Purpose |
|---|---|
| /batch | Parallel changes across codebase (one worktree per unit) |
| /simplify | Review changed code for quality |
| /loop | Run a prompt on recurring interval |
| /debug | Debug logging and troubleshooting |
| /claude-api | Claude API/SDK reference |

## Built-in Tools

Do NOT create agents or skills that merely wrap these:

| Tool | Purpose |
|---|---|
| Agent | Spawn subagents |
| Bash | Execute shell commands |
| Edit | Targeted file edits |
| Write | Create/overwrite files |
| Read | Read files (text, images, PDFs, notebooks) |
| Glob | File pattern matching |
| Grep | Content search (ripgrep) |
| LSP | Code intelligence (go-to-def, references, hover, symbols) |
| WebFetch | Fetch URL content |
| WebSearch | Web search |
| TaskCreate/Update/List/Get | Task management |
| EnterPlanMode/ExitPlanMode | Plan mode workflow |
| EnterWorktree/ExitWorktree | Isolated git worktrees |
| CronCreate/Delete/List | In-session scheduled tasks |
| Skill | Execute a skill |
| ToolSearch | Load deferred tools |
| NotebookEdit | Modify Jupyter notebooks |

## Built-in Subagent Types

Do NOT create agents that duplicate these roles:

| Type | Model | Purpose |
|---|---|---|
| Explore | haiku | Fast codebase search, file/symbol discovery (READ-ONLY) |
| Plan | inherited | Codebase research for planning (READ-ONLY) |
| general-purpose | inherited | Complex multi-step operations, full tool access |
| Bash | inherited | Shell command execution in separate context |

Custom agents ADD specialization on top of these types. Example: a "security-reviewer" adds security expertise to `general-purpose`, not replacing it.

## Built-in Hook Events (25)

These lifecycle events are available for hooks. Do NOT create custom event systems:

**Session:** SessionStart, SessionEnd, InstructionsLoaded
**User Input:** UserPromptSubmit
**Tools:** PreToolUse, PostToolUse, PostToolUseFailure, PermissionRequest
**Agents:** SubagentStart, SubagentStop, Stop, StopFailure, TeammateIdle
**Tasks:** TaskCreated, TaskCompleted
**Files:** FileChanged, CwdChanged, ConfigChange
**Worktrees:** WorktreeCreate, WorktreeRemove
**Compaction:** PreCompact, PostCompact
**MCP:** Elicitation, ElicitationResult
**UI:** Notification

**Hook handler types:** command (shell), http (POST endpoint), prompt (LLM eval), agent (subagent)

## Agent Teams (Experimental)

Available when `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`:

| Feature | Tool |
|---|---|
| Create/delete teams | TeamCreate, TeamDelete |
| Inter-agent messaging | SendMessage (direct + broadcast) |
| Remote triggers | RemoteTrigger |
| Task coordination | TaskCreate with dependencies |

Do NOT build custom inter-agent communication if Agent Teams is enabled.

## Other Built-in Features

Do NOT replicate:
- **Auto-memory** — Saves context to MEMORY.md automatically
- **Context compaction** — Auto-compacts at ~95% capacity
- **Git worktrees** — --worktree flag for isolated parallel work
- **Checkpointing** — /rewind restores conversation and code
- **Remote control** — Bridge sessions to claude.ai/code
- **Sandboxing** — Isolate file system and network
- **Plugin system** — Full plugin packaging and marketplace
- **MCP support** — Full client/server with registry, OAuth, elicitation

## What IS Worth Generating

These add value beyond built-ins:
- **Domain-specific agents** — Security reviewer for auth code, API designer for REST APIs
- **Workflow skills** — Multi-step processes specific to the project
- **Safety hooks** — Destructive command guards, edit freeze zones
- **Quality hooks** — Auto-formatting, type checking after edits
- **Project rules** — CLAUDE.md with conventions, delegation logic
- **Orchestration** — Multi-agent pipelines for complex workflows
