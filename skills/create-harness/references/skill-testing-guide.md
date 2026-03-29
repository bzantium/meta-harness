# Skill Testing Guide

How to validate that generated skills trigger correctly and produce expected results.

---

## 1. Trigger Validation

Every generated skill must be tested with two query sets:

### Should-Trigger Queries

Write 3-5 phrases that MUST activate the skill. Include:
- Direct command: "run the deploy pipeline"
- Indirect request: "ship this to production"
- Contextual: "we need to get this live"
- Abbreviation/slang: "deploy it", "push to prod"

### Should-NOT-Trigger Queries

Write 2-3 phrases that must NOT activate the skill. Include:
- Adjacent but different: "review the deploy config" (review, not deploy)
- Ambiguous: "check the pipeline" (could be CI, not deploy)

### How to Test

For each query, mentally evaluate: would the skill description match this input?

| Query | Expected | Triggered? | Result |
|---|---|---|---|
| "deploy to production" | YES | YES | PASS |
| "review deploy scripts" | NO | NO | PASS |
| "ship it" | YES | NO | **FAIL — description needs "ship" as trigger phrase** |

If any FAIL: rewrite the skill description to fix it.

---

## 2. Trigger Collision Detection

Check all generated skills pairwise for overlap:

1. List every skill's trigger phrases side by side
2. Identify phrases that could match multiple skills
3. For collisions: make descriptions more specific, or merge the skills

**Common collision patterns:**
- "review" — could match code-review skill AND security-review skill
- "test" — could match test-runner skill AND QA skill
- "check" — vague enough to match almost anything

**Fix:** Add domain qualifiers. "Review code for logic bugs" vs "Review code for security vulnerabilities."

---

## 3. With-Skill vs Without-Skill Comparison

For each generated skill, ask: what does the user gain that Claude wouldn't do by default?

| Skill | Without skill | With skill | Value added? |
|---|---|---|---|
| deploy-pipeline | User manually lists deploy steps each time | Skill encodes the full pipeline | YES |
| code-search | Claude uses built-in Explore agent | Skill wraps the same Explore agent | **NO — remove** |

If "Value added?" is NO, the skill should not exist.

---

## 4. Dry-Run Checklist

Before finalizing, verify each skill against:

- [ ] Description contains 5+ trigger phrases
- [ ] Description ends with "Use this skill whenever..." or "Triggers on:"
- [ ] No trigger collision with other skills
- [ ] No trigger collision with built-in slash commands
- [ ] Workflow steps reference tools that exist
- [ ] Workflow references agents that exist in the harness
- [ ] Body under 500 lines
- [ ] Heavy content delegated to references/
