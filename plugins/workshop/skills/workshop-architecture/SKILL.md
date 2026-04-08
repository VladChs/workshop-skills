---
name: workshop-architecture
description: Use after brainstorm/PRD is complete and before scaffolding or writing app code in a "Build Your Idea" workshop project. Loads universal backend/frontend architecture rules that Claude must follow as the default structure.
---

# Workshop Architecture

A small, opinionated, universal reference for structuring workshop apps. Read this BEFORE making any structural decision (folder layout, layer split, where logic goes, where state lives, where errors are caught, what crosses module boundaries).

## When to use

Trigger on ALL of:
1. The current project is a "Build Your Idea" workshop project (or the user explicitly invokes this skill).
2. Brainstorming and the PRD/spec phase are complete.
3. You are about to scaffold code, create new modules, or make a structural decision.

Do NOT trigger on: trivial edits to existing code, doc-only changes, or single-file scripts under ~50 lines.

## How to use

1. Identify whether the project has a backend, a frontend, or both.
2. Read the matching reference doc(s) end to end:
   - Backend → `references/backend.md`
   - Frontend → `references/frontend.md`
3. Treat the rules in those docs as the default structure. Apply ALL rules unless the spec explicitly overrides one — and if it does, note the override in the implementation.
4. The reference describes ROLES and RULES, not stacks. Pick a stack appropriate for the project; the rules apply regardless.
5. When in doubt about a rule, follow it. The whole point is to remove decision wobble.

## Situational patterns (NOT in the references)

If the spec reveals any of these triggers, STOP and surface it to the user before inventing a pattern:
- Plugin/extensibility system needed
- RBAC / multi-actor permissions
- Audit trail or event-sourced history
- Optimistic locking, soft delete, partial-update sentinels
- Streaming / async-generator interfaces
- Long-running services with retries, backoff, graceful shutdown
- Layered config hierarchy (managed > user > project > local)
- Domain events for cross-module communication
- Idempotency keys for external-facing writes

These are intentionally OMITTED from the universal references. They will live in a future specialized appendix. Until then, ask the user how to proceed if any are required.

## Anti-pattern

Do not "summarize" the references and skip reading them. Do not pick a few rules you remember. Read the file each time the skill triggers — the rules are short, the cost is low, and consistency is the entire value of the skill.
