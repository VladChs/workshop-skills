# Workshop Skills

A Claude Code marketplace plugin that turns any idea into a live, deployed app in roughly four hours — with structure, taste, and a pipeline you can see.

It wraps the [`superpowers`](https://github.com/anthropics/claude-plugins-public) plugin with workshop-tuned rules: opinionated brainstorming, universal architecture patterns, pragmatic testing, frontend aesthetics, and a one-shot Vercel deploy that wires GitHub auto-redeploy on the way out.

## What you'll experience

You start a fresh Claude Code session in an empty directory and say *"start the workshop build."* Claude takes you through five visible phases:

1. **Brainstorm.** Who is this for? What pain are they feeling? What's the *one thing* that makes this valuable? What's the 4-hour slice that proves it? Claude refuses to let you skip these.
2. **Plan.** Claude reads the universal architecture rules and writes an implementation plan whose file paths, layers, and module boundaries already encode them.
3. **Execute.** A fresh subagent per task implements with TDD on the core value path and pragmatic testing elsewhere. If your project has a UI, Claude commits to an aesthetic direction once and follows it through every component.
4. **Review.** Code is reviewed against the 25 architecture rules. Production-hardening nags become notes, not blockers — this is a 4-hour build, not a launch.
5. **Finish.** Claude writes a `RECAP.md`, then offers to deploy to Vercel. If you say yes, you walk away with a live URL and auto-deploy already wired so every future push redeploys.

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

## Pre-reqs

| Tool | Required for | Install |
|---|---|---|
| Claude Code | everything | [docs.anthropic.com/claude-code](https://docs.anthropic.com/en/docs/claude-code) |
| `superpowers` plugin | the pipeline wraps it | `/plugin marketplace add anthropics/claude-plugins-official` then `/plugin install superpowers@claude-plugins-official` |
| `frontend-design` plugin | UI projects (Extension H) | `/plugin install frontend-design@claude-plugins-official` |
| `vercel` CLI | Extension I (deploy) | `npm i -g vercel` then `vercel login` |
| `gh` CLI authenticated | Extension I (GitHub repo creation) | `brew install gh && gh auth login` |

The Vercel CLI and `gh` are optional — they only matter if you want the deploy moment at the end. The pipeline runs without them.

## Install

```
/plugin marketplace add VladChs/workshop-skills
/plugin install workshop@workshop-marketplace
```

## Use

In a fresh Claude Code session, in a clean directory where you want your project to live, say:

> *"Start the workshop build."*

Claude invokes `workshop-pipeline` and asks the brainstorm questions in order. Answer them honestly — the discipline is the value. Then let the pipeline run.

If you want the Vercel deploy at the end, make sure `vercel` and `gh` are installed and authed before the workshop starts. Extension I will detect whether your build fits Vercel, prompt you, and (if you say yes) wire everything up.

### What to expect at each phase

| Phase | Duration target | What Claude does | What you do |
|---|---|---|---|
| Brainstorm | ~45 min | Asks user/pain/value/scope/aesthetic | Answer truthfully; let scope get cut |
| Plan | ~10 min | Reads architecture rules, writes a structured plan | Skim, approve |
| Execute | ~2.5 hr | Spawns subagents per task, TDD on the core path | Watch, intervene only if stuck |
| Review | ~10 min | Checks code against the 25 rules | Skim, accept blockers |
| Finish | ~15 min | Writes `RECAP.md`, offers Vercel deploy | Say yes to the deploy, watch the URL appear |

Total: about four hours, end to end.

## Update

```
/plugin update workshop@workshop-marketplace
```

## Troubleshooting

**Plugin won't install / "invalid manifest" error.** Update Claude Code itself (`claude --version`), remove the cached marketplace (`/plugin marketplace remove VladChs-workshop-skills`), then re-add and re-install. Older Claude versions accepted looser manifests; the current schema is strict.

**`git push` fails with "Permission denied (publickey)".** Run `gh auth setup-git` once to use your `gh` token as the credential helper. The Vercel deploy step depends on `git push` working.

**Claude doesn't auto-trigger `workshop-pipeline` after `/clear`.** Re-invoke explicitly: *"Use the workshop-pipeline skill."* For persistent triggering across sessions, drop a one-line `CLAUDE.md` in your project root that says *"If the user says 'start the workshop build,' invoke `workshop-pipeline`."*

**Extension I says "doesn't fit Vercel" and you think it should.** The detection is conservative. Re-read its reasoning in the response and decide whether to deploy yourself with `vercel --prod` directly. The recap will still record the manual-deploy path.

**FastAPI on Vercel runs out of memory or times out.** Hobby tier serverless functions have a 10s execution limit and 50 MB unzipped function size. If your build needs more, deploy to Railway or Fly instead — Extension I will not try to work around platform limits.

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

Issues and PRs welcome. The fastest way to help: run a workshop build, hit a rough edge, and open an issue with the spec, the recap, and the rough edge.
