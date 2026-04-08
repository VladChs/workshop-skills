# Brainstorm Overlay — Product-Design Rules

> **Status:** v1 baseline. Will be refined by a dedicated product-design brainstorm. Until then, these rules ship as the workshop-pipeline Extension A defaults.

These rules are layered on top of `superpowers:brainstorming` whenever `workshop-pipeline` Extension A is in effect. Read this entire file at the start of the brainstorm phase. Apply every rule. Do not skip rules to save time — the discipline is the value.

## The seven rules

### 1. User and pain first

Before any solution talk, force the participant to state two things:

- **Who the user is** — a specific person or a tightly-defined group, not "everyone."
- **What pain they feel** — a concrete moment of friction, not an abstract problem.

Reject "I want to build X" framings. Reframe as: *"What is the moment of pain that makes someone reach for X?"*

### 2. The one thing

Force articulation of **the single sentence that makes this valuable.** One sentence. Specific. Concrete enough that you could build only that and the result would still be useful.

If the participant cannot name it, the brainstorm is not done. Keep asking.

This sentence becomes the "core value path" referenced by Extension G — the only path that gets iron-law TDD treatment during execution.

### 3. 4-hour scope cut

Propose the minimum viable slice that can be built in roughly four hours **and** demonstrate the one thing from rule 2. Aggressively remove everything else.

Default to cutting:
- Authentication (use a hardcoded user or skip auth entirely)
- User management (one user is fine)
- Settings, preferences, profile pages
- Admin views
- Onboarding flows
- Multi-tenancy
- Internationalization
- Accessibility audits beyond keyboard navigation
- Analytics, telemetry, observability
- Email, notifications
- Pagination beyond a hardcoded limit
- Search beyond a flat filter
- Anything described as "we'll need this eventually"

If the participant pushes back on a cut, ask: *"Does this exist for the one thing in rule 2, or for a different reason?"* If different, cut it.

### 4. Why not an existing tool

Force the question: *"Why can't you use $EXISTING_THING for this?"*

Pick a real existing tool — Notion, Airtable, a Google Form + Sheet, an existing SaaS in the space, a no-code builder. Name it specifically. Do not let the participant answer in the abstract.

Three valid answers:
- *"I looked at $EXISTING_THING and it can't do X, where X is the one thing from rule 2."* — proceed with the build, document the gap in the spec.
- *"I haven't looked."* — pause, look, return.
- *"$EXISTING_THING does it, but I want to learn by building it myself."* — proceed, but be honest in the spec that the goal is learning, not differentiation.

Any other answer is a signal to keep digging.

### 5. Visual companion for frontend projects

When the base `superpowers:brainstorming` skill offers its visual companion, **accept it** for any project with a UI. Workshop frontends benefit from mockups, layout comparisons, and side-by-side visual options during brainstorm.

For text-only projects (CLI tools, libraries, backend services), decline the visual companion as usual.

### 6. Aesthetic intent (UI projects only)

Capture a **1-line aesthetic intent** in the spec. Examples:

- "brutalist editorial"
- "soft pastel toy-like"
- "industrial utilitarian"
- "retro-futuristic terminal"
- "luxury minimalist"
- "playful zine"

Without this line, frontends default to vanilla AI output (Inter font, purple gradients, generic shadcn shapes). The line forces commitment to a direction.

This line is consumed by Extension H at execution time, when the `frontend-design` skill is invoked with the intent as input.

### 7. Commit guidance

Tell the participant once, near the end of the brainstorm:

> *"Commits should be short, present tense, one logical change. Don't worry about strict conventions."*

The wrapper does not enforce a commit-message format beyond this. Don't lecture about Conventional Commits or sign-offs.

## What the spec must contain

The spec produced by brainstorming MUST contain, in any reasonable order:

1. **User** — specific person or tight group (rule 1)
2. **Pain** — concrete moment of friction (rule 1)
3. **The one thing** — single value sentence (rule 2)
4. **4h-scoped feature list** — only the cuts that survive rule 3
5. **Why not $EXISTING_THING** — the answer to rule 4 with the tool named
6. **Aesthetic intent** — one line, only for UI projects (rule 6)

If any of these are missing when brainstorming concludes, the brainstorm is not done.

## What this overlay is NOT

- Not a replacement for `superpowers:brainstorming`. The base skill still runs; this overlay layers on top.
- Not a market-research exercise. Don't ask for TAM, competitor analysis beyond rule 4, pricing, or growth strategy.
- Not a design-doc template. The spec format is whatever `superpowers:brainstorming` produces; this overlay only constrains the *content* it must contain.
- Not a time-box enforcer. The 4-hour figure in rule 3 is a scope-setting heuristic, not a stopwatch.
