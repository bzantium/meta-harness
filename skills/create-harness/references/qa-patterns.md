# QA Patterns

Common bug categories and detection strategies across project types. Use when generating QA agents or review workflows.

---

## 1. Cross-Boundary Bugs

Bugs at the interface between two modules — the most common source of integration failures.

**Detection strategy:** Read both sides of an interface simultaneously and compare.

| Boundary | What to Check | Common Bug |
|---|---|---|
| API endpoint ↔ Frontend call | Request/response shape match | Frontend sends `userId`, API expects `user_id` |
| Database schema ↔ ORM model | Column types and nullability | ORM allows null, DB column is NOT NULL |
| Config file ↔ Code that reads it | Key names and default values | Code reads `DB_HOST`, config has `DATABASE_HOST` |
| Type definition ↔ Runtime usage | Optional vs required fields | Type says optional, code accesses without check |
| API v1 ↔ API v2 | Backward compatibility | v2 removes field that v1 clients still send |

**QA agent instruction:** "For each modified module, identify its consumers and read both sides. Report any shape mismatch, naming inconsistency, or nullability disagreement."

---

## 2. Silent Failure Bugs

Code that fails without error — the hardest bugs to catch because tests pass.

| Pattern | Example | Detection |
|---|---|---|
| Empty catch block | `catch (e) {}` | Grep for empty catch/except blocks |
| Swallowed error return | `result, _ := fn()` (Go) | Grep for `_ :=` or `_ =` on error returns |
| Default fallback hides bug | `config.get("key", "default")` | Check if default masks missing required config |
| Fire-and-forget async | `Promise` without `.catch()` | Grep for unhandled promises |
| Logging instead of failing | `console.error(e); return null` | Check if callers handle null return |

**QA agent instruction:** "Search for error suppression patterns. For each one found, evaluate whether the suppression is intentional (documented) or accidental (no comment explaining why)."

---

## 3. State Consistency Bugs

Data that gets out of sync between components.

| Scenario | What to Check |
|---|---|
| Cache ↔ Database | Cache invalidation on every write path |
| UI state ↔ Server state | Optimistic update has rollback on failure |
| File system ↔ Database | Transaction covers both or has cleanup |
| Environment variables ↔ Docs | Every env var used in code is documented |
| Package.json ↔ Lock file | Lock file is up to date after dependency changes |

**QA agent instruction:** "Identify all state pairs (cache/DB, UI/server, etc.). For each pair, trace the write path and verify both sides update atomically or have compensation logic."

---

## 4. Incremental QA Strategy

Don't run QA once at the end. Run after each module completion:

```
Module A complete → QA checks A's interfaces
Module B complete → QA checks B's interfaces + A↔B boundary
Module C complete → QA checks C's interfaces + B↔C + A↔C boundaries
```

**Why:** Bug count at boundaries grows quadratically with module count. Early detection prevents compounding.

**Implementation:** After each agent completes a module, spawn QA agent with:
- The completed module's public API/exports
- All existing modules that consume or are consumed by it
- Instruction: "Read both sides of every boundary. Report mismatches."

---

## 5. Domain-Specific Checklists

### Web Applications
- [ ] Auth routes return 401/403, not 500, on invalid tokens
- [ ] CORS config matches actual frontend origins
- [ ] CSP headers present on pages with user content
- [ ] Form validation exists on both client AND server

### API Backends
- [ ] All endpoints have input validation
- [ ] Pagination has a maximum page size
- [ ] Error responses follow consistent format
- [ ] Rate limiting on public endpoints

### CLI Tools
- [ ] `--help` text matches actual behavior
- [ ] Exit codes are non-zero on failure
- [ ] Stdin/stdout/stderr used correctly (data to stdout, errors to stderr)
- [ ] Ctrl+C handled gracefully (cleanup runs)
