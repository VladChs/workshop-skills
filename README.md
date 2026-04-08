# Workshop Skills

A Claude Code marketplace plugin that turns any idea into a live, deployed app in roughly four hours — with structure, taste, and a pipeline you can see.

It wraps the [`superpowers`](https://github.com/anthropics/claude-plugins-public) plugin with workshop-tuned rules: opinionated brainstorming, universal architecture patterns, pragmatic testing, frontend aesthetics, and a one-shot Vercel deploy that wires GitHub auto-redeploy on the way out.

## Pre-reqs

| Tool | Required for | Install |
|---|---|---|
| Claude Code | everything | [docs.anthropic.com/claude-code](https://docs.anthropic.com/en/docs/claude-code) |
| `superpowers` plugin | the pipeline wraps it | `/plugin marketplace add anthropics/claude-plugins-official` then `/plugin install superpowers@claude-plugins-official` |
| `frontend-design` plugin | UI projects (Extension H) | `/plugin install frontend-design@claude-plugins-official` |
| `vercel` CLI | Extension I (deploy) | `npm i -g vercel` then `vercel login` |
| `gh` CLI authenticated | Extension I (GitHub repo creation) | `brew install gh && gh auth login` |

The Vercel CLI and `gh` are optional — they only matter if you want the deploy moment at the end.

## Install

```
/plugin marketplace add VladChs/workshop-skills
/plugin install workshop@workshop-marketplace
```

## Use

In a fresh Claude Code session, in a clean directory where you want your project to live, say:

> *"Start the workshop build."*

Claude invokes `workshop-pipeline` and walks you through the phases.

## Update

```
/plugin update workshop@workshop-marketplace
```

---

## What's inside

Two skills, designed to be used together:

### `workshop-pipeline`
The entry-point skill. Wraps the standard `superpowers` pipeline (brainstorming → writing-plans → subagent-driven-development → code review → finishing) and injects seven workshop extensions:

- **A — Brainstorm overlay**: forces *user, pain, the-one-thing, 4h-cut, why-not-existing-tool,* and (for UI projects) a 1-line aesthetic intent. Captures commit guidance.
- **B — Architecture auto-load**: reads the universal patterns at plan-writing time so the plan itself encodes correct structure.
- **D — Workshop-tuned review**: blockers come from the 25 architecture rules; production-hardening concerns are notes, not blockers.
- **F — Simplified finishing**: replaces the standard PR/merge ceremony with a five-section `RECAP.md` (what was built, rules applied, what was cut, test honesty, next steps).
- **G — Pragmatic testing**: TDD is iron-law on the core value path; auxiliary code is encouraged but not blocking. A manual smoke test gates the recap.
- **H — Frontend design**: invokes the `frontend-design` skill once at execution start to commit to typography/color/motion/layout, then every component honors that direction.
- **I — Vercel deploy** (optional, opt-in): detects whether the build fits Vercel, offers a one-shot `vercel link` + `vercel git connect` + `vercel --prod`, and writes the live URL to `RECAP.md`.

### `workshop-architecture`
A reference skill loaded by the pipeline at plan-writing time. Two stack-agnostic markdown docs (`backend.md`, `frontend.md`) covering 25 universal patterns: strict downward layering, services as the only mutation/query boundary, parse-don't-validate, value objects at boundaries, ports & adapters, functional core / imperative shell, anti-corruption layers, dependency injection without globals, discriminated unions with `assertNever` exhaustiveness, custom error hierarchies, and the rest. Sourced from production codebases plus DDD / Clean / Hexagonal literature.

The reference is intentionally narrow. Situational patterns (RBAC, audit trails, plugin systems, streaming, queues, idempotency keys) are deliberately *not* in the universal core — when the spec triggers them, the skill stops and asks rather than silently inventing.

## The pipeline at a glance

```
 START
   │
   ▼
┌──────────────────────────────┐
│ superpowers:brainstorming    │   ← Extension A (product-design overlay)
└──────────────────────────────┘
   │  produces: spec doc
   ▼
┌──────────────────────────────┐
│ superpowers:writing-plans    │   ← Extension B (architecture auto-load)
└──────────────────────────────┘
   │  produces: implementation plan
   ▼
┌──────────────────────────────────────┐
│ superpowers:subagent-driven-         │   ← Extensions G + H apply during exec
│ development                          │
└──────────────────────────────────────┘
   │  produces: committed code
   ▼
┌──────────────────────────────┐
│ superpowers:requesting-      │   ← Extension D (workshop-tuned review)
│ code-review                  │
└──────────────────────────────┘
   │
   ▼
┌──────────────────────────────┐
│ Simplified finishing         │   ← Extension F (RECAP.md)
└──────────────────────────────┘
   │
   ▼
┌──────────────────────────────┐
│ Vercel deploy (optional)     │   ← Extension I (live URL + auto-redeploy)
└──────────────────────────────┘
```

## Status

Current version: **0.2.0**

| Component | Status |
|---|---|
| `workshop-architecture` (25 rules) | stable |
| `workshop-pipeline` Extensions B, D, F, G, H | stable |
| `workshop-pipeline` Extension A (product-design overlay) | **v1 baseline** — will be refined by a dedicated product-design brainstorm |
| `workshop-pipeline` Extension I (Vercel deploy) | shipped, lightly tested |

## Influences

- [`superpowers`](https://github.com/anthropics/claude-plugins-public) — the wrapped pipeline and most of the discipline
- [`ralph`](https://github.com/snarktank/ralph) — autonomous-execution model (sidecar pointer in Extension F)
- The Claude Code patterns workshop — eight cross-cutting patterns in `workshop-architecture`
- DDD (Eric Evans), Clean Architecture (Uncle Bob), Hexagonal (Alistair Cockburn), Functional Core / Imperative Shell (Gary Bernhardt), *Parse, Don't Validate* (Alexis King)

## License

MIT — see [LICENSE](LICENSE).

## Contributing

Issues and PRs welcome.
