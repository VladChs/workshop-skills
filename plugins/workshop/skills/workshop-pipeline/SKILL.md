---
name: workshop-pipeline
description: Use at the start of a "Build Your Idea" workshop build (4-hour scope). Runs the full superpowers pipeline (brainstorm → plan → execute → review → finish) with workshop-tuned extensions for product-design discipline, universal architecture rules, pragmatic testing, and frontend aesthetics.
---

# Workshop Pipeline

Single entry point for a 4-hour workshop build. Wraps the standard superpowers pipeline and injects workshop-specific rules at each stage. The phases stay visible so the participant learns the flow.

## When to use

Trigger when ALL of:
1. The current project is a "Build Your Idea" workshop project (or the user explicitly invokes this skill).
2. The participant is about to start a fresh build.
3. No spec or plan exists yet for the build.

Do NOT trigger on: continuing existing work, doc-only edits, or "fix this bug" requests in an existing workshop project.

## Pre-reqs

Verify the following are installed before the pipeline starts. If any are missing, stop and tell the participant which `/plugin install` command to run.

- `superpowers` (claude-plugins-official) — required, the pipeline wraps it.
- `workshop-architecture` — required, ships in the same plugin as this skill.
- `frontend-design` (claude-plugins-official) — required only for projects with a UI.
- `vercel` CLI (`npm i -g vercel`) — **optional**, only required if the participant wants Extension I to deploy at the end. Authenticate with `vercel login` before the workshop starts.
- `gh` CLI authenticated — required for Extension I (to create a GitHub repo if one does not exist) and already implied by the marketplace install.

## Pipeline map

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
│ superpowers:writing-plans    │   ← Extension B (workshop-architecture
│                              │      auto-loaded at plan time)
└──────────────────────────────┘
   │  produces: implementation plan
   ▼
┌──────────────────────────────────────┐
│ superpowers:subagent-driven-         │   (default executor)
│ development                          │   ← Extensions G + H apply during exec
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
│ Simplified finishing         │   ← Extension F (replaces finishing-a-
│ (RECAP.md)                   │      development-branch)
└──────────────────────────────┘
   │
   ▼
┌──────────────────────────────┐
│ Vercel deploy (optional)     │   ← Extension I (opt-in, Vercel-fit
│                              │      detected, wires GitHub auto-deploy)
└──────────────────────────────┘
```

Stage handoff: when one stage completes, immediately invoke the next (`superpowers:brainstorming` → `superpowers:writing-plans` → `superpowers:subagent-driven-development` → `superpowers:requesting-code-review` → Extension F). Do not wait for the participant unless a stage's own skill requires it.

## Extension A — Brainstorm: product-design overlay

Tag: `v1 — revisit after dedicated product-design brainstorm`

Apply during `superpowers:brainstorming`. Add these rules to the base brainstorming flow:

1. **User and pain first.** Before any solution talk, force the participant to state who the user is and what pain they feel. No "I want to build X" without this.
2. **The one thing.** Force articulation of the single sentence that makes this valuable. If the participant cannot name it, the brainstorm is not done.
3. **4-hour scope cut.** Propose the minimum viable slice that can be built in 4 hours and demonstrate the one thing. Aggressively remove everything else.
4. **Why not an existing tool.** Force the question: "Why can't you use $EXISTING_THING for this?" If the answer is "I haven't looked," that is a signal to look first.
5. **Visual companion for frontend projects.** When the base brainstorming skill offers its visual companion, accept it. Workshop frontends benefit from mockups during brainstorm.
6. **Aesthetic intent (UI projects only).** Capture a 1-line aesthetic intent in the spec — examples: "brutalist editorial," "soft pastel toy-like," "industrial utilitarian." Without this line, frontends default to vanilla AI output. This line is consumed by Extension H during execution.
7. **Commit guidance.** Tell the participant: commits are short, present tense, one logical change. The wrapper does not enforce a commit-message format beyond this.

The spec produced by brainstorming MUST contain: user, pain, the-one-thing sentence, 4h-scoped feature list, "why not existing tool" answer, and (if UI) the aesthetic intent line.

## Extension B — Writing plans: workshop-architecture auto-load

Apply during `superpowers:writing-plans`. Before writing any task in the plan, you MUST:

1. Identify whether the project has a backend, a frontend, or both.
2. Read `skills/workshop-architecture/references/backend.md` and/or `skills/workshop-architecture/references/frontend.md` accordingly. (These ship in the same plugin as this skill.)
3. Ensure task file paths, layer splits, module boundaries, and error-handling locations in the plan reflect the rules in those references. The plan itself encodes the universal patterns so execution does not need to re-apply them.

If the spec reveals a situational trigger listed in `workshop-architecture/SKILL.md` (plugin system, RBAC, audit, streaming, etc.), STOP and ask the participant how to proceed. Do NOT silently invent the pattern.

## Extension D — Review: workshop-tuned checklist

Apply during `superpowers:requesting-code-review`. The review checklist is the 25 universal rules from `workshop-architecture`.

- **Blocker:** any violation of rules 1–25. The review does not pass until fixed.
- **Note, not blocker:** production-hardening concerns — logging completeness, observability, scale, caching, rate limiting, auth hardening beyond the rules. Flag for the participant's follow-up but do not block the workshop build.
- **Out of scope:** stylistic preferences not covered by the rules.

## Extension F — Simplified finishing: workshop recap

REPLACES `superpowers:finishing-a-development-branch`. Do NOT invoke that skill. Instead, produce a recap at the workshop project root.

Create `RECAP.md` with these five sections:

1. **What was built** — one paragraph.
2. **Which rules were applied** — the numbered list of `workshop-architecture` rules that shaped the build.
3. **What was cut and why** — features explicitly descoped during Extension A's 4h cut.
4. **Test coverage honesty** — what is tested (the core value path, per Extension G), what is not (auxiliary glue).
5. **Next steps** — the top 3 things the participant could do next, including the Ralph sidecar pointer (see below).

No git push, no PR, no branch cleanup.

## Extension G — Pragmatic testing for workshop scope

Apply throughout execution and review.

1. **The core value path is TDD-mandatory.** "Core value path" = the one-sentence answer from Extension A rule 2. Tests for this path exist, are written first (TDD), and pass.
2. **Auxiliary code is TDD-encouraged but not blocking.** Glue code, scaffolding, simple CRUD endpoints, presentational components — tests are welcome but not required to ship.
3. **Manual smoke test before finishing is required.** Before Extension F runs, walk the app's core flow end-to-end manually. Failures here block the recap.
4. **Recap reports honest coverage** (Extension F item 4).

Interaction with `superpowers:test-driven-development`: the TDD skill is still invoked, but its iron law ("no production code without a failing test first") is scoped to the core value path. For auxiliary code, TDD is recommended, not enforced.

## Extension H — Frontend design

Apply to any project with a UI.

1. **Brainstorm captures intent** (Extension A rule 6 records the 1-line aesthetic intent in the spec).
2. **Execution invokes `frontend-design`.** Before the first UI component is implemented, invoke the `frontend-design` skill (claude-plugins-official) with the spec's aesthetic intent as input. Commit to a direction: typography, color, motion, spatial composition.
3. **Subsequent components honor the committed direction.** Once the direction is set, every UI component follows it. Do not re-derive aesthetic choices per component.
4. **Pre-req:** `frontend-design` must be installed (see Pre-reqs above).

Interaction with `workshop-architecture/frontend.md`:
- `workshop-architecture/frontend.md` controls **structure** (data direction, component tiers, state homes, error boundaries, types).
- `frontend-design` controls **aesthetics** (typography, color, motion, layout).
- Both apply to every component. Structure rules are blockers in review (Extension D); aesthetic rules are subjective and not enforced beyond "the direction was committed and followed consistently."

## Extension I — Vercel deploy (optional)

Apply ONLY if the participant opts in. Runs after Extension F has written `RECAP.md`. The recap always exists; the Live URL is appended on top only when the deploy succeeds. The pipeline ends here.

### Detection — classify the project before prompting

Read the spec and the project structure. Pick exactly one bucket.

**Green — fits Vercel cleanly.** ALL of:
- Frontend uses a Vercel-native framework (Next, Astro, SvelteKit, Nuxt, Vite + React/Vue/Svelte) OR is a static site.
- Backend (if present) is Python or Node serverless functions in `api/` (FastAPI mounted as ASGI in `api/index.py`, Express, plain handlers).
- DB (if present) is Vercel Postgres, Vercel KV, or an external managed service (Supabase, Neon, Upstash, PlanetScale).
- Spec/code does NOT mention: websockets, long-running SSE, background workers, queues, cron jobs, persistent local filesystem, audio/video processing, requests longer than 10s, native binary deps over 50 MB.

**Red — does NOT fit Vercel.** ANY of:
- Spec or code uses websockets, queues, cron, background workers, or persistent local FS.
- Backend is a long-running process (uvicorn started as a server, Django with file-backed sessions, Celery worker, etc.).
- Native binary deps exceed Vercel's 50 MB unzipped function limit.
- Sync work per request exceeds the hobby-tier 10s limit.

**Yellow — unsure.** None of the above apply but you are not confident (e.g., unrecognized backend library). Treat as "ask the participant explicitly."

### Three branches

**Green path:**

1. Prompt the participant: *"Your build looks Vercel-ready. Deploy it now? It takes ~2 minutes and you'll get a live URL with auto-deploy wired up. (yes / no)"*
2. If `no` → append `deploy skipped by user` to RECAP next-steps. Stop.
3. If `yes` → run the **Deploy flow** below.

**Red path:**

1. Do NOT prompt.
2. Append to RECAP next-steps: *"This build can't be auto-deployed to Vercel because [specific reason]. To deploy yourself, see Vercel docs, Railway, or Fly.io."* The reason MUST be specific (e.g., "uses websockets," "needs persistent local filesystem"), not generic.
3. Stop.

**Yellow path:**

1. Prompt with the uncertainty: *"I'm not 100% sure this fits Vercel because of [specific reason]. Want me to try anyway? (yes / no)"*
2. If `no` → append `deploy skipped due to uncertainty` to RECAP next-steps. Stop.
3. If `yes` → run the **Deploy flow** below. If any step fails, fall through to the Red branch's recap message with the failure reason.

### Deploy flow

1. **Verify pre-reqs.** Check that `vercel` CLI is installed (`vercel --version`) and authed. If missing, tell the participant: *"Install with `npm i -g vercel` then run `vercel login`. Reply when ready."* Wait. Do NOT install the CLI yourself.

2. **Ensure the project is on GitHub.** Run `git remote -v`. If no GitHub remote exists, ask: *"This deploy needs a GitHub repo so Vercel can auto-redeploy on push. Want me to create one? (yes / no)"*
   - `yes` → run `gh repo create <project-name> --public --source . --push`.
   - `no` → append `deploy skipped: no GitHub repo` to RECAP next-steps and stop.

3. **Detect environment variables.** Grep the spec and code for `process.env.X`, `os.environ['X']`, `os.getenv('X')`, and any `.env.example` entries. Build a list of required env var names (NOT values).

4. **Append env-var note to RECAP** (only if any were found). Add to next-steps: *"Before this deploy works at runtime, set these env vars in your Vercel dashboard: [list]. The build will deploy but the app may fail at runtime with a clear error until they're set."*

5. **`vercel link`** — creates a Vercel project linked to the local directory. Writes `.vercel/project.json`. First run prompts the participant for project name and scope; defaults are fine.

6. **`vercel git connect`** — wires the Vercel project to the GitHub repo for auto-deploy. First run triggers a browser flow to authorize Vercel's GitHub app. Wait for the participant to complete the browser flow.

7. **`vercel --prod`** — runs the first production deploy. Vercel CLI handles framework detection automatically.

8. **Capture the deployed URL** from CLI output. Use the stable `https://<project>.vercel.app` URL, not the per-deploy hash URL.

9. **Append to `RECAP.md` as the top header** (above the existing five sections):

   ```markdown
   **Live URL:** https://<project>.vercel.app

   _Auto-deploy is wired: every push to `main` redeploys; every PR creates a preview URL._

   ---
   ```

10. **If any step fails**, capture stderr, write a one-paragraph failure note plus manual-deploy instructions into RECAP next-steps, and stop. Do NOT retry automatically.

### Out of scope for Extension I

- Custom domains
- Vercel Postgres / Neon provisioning (note as a manual next step only)
- Pushing env var values
- Branch protection rules
- Preview-deploy comment automation
- Multi-environment deploys

## Ralph sidecar pointer

For participants who finish early and want to see autonomous execution: install Ralph from `https://github.com/snarktank/ralph`, convert your spec to a `prd.json` via Ralph's `/ralph` skill, and run `ralph.sh` to watch a fresh Claude instance grind through stories autonomously. Out of scope for the main pipeline — recommend it in Extension F's "next steps" only.

## Anti-patterns

Do NOT:
- Skip a phase (brainstorm → plan → execute → review → finish is a fixed order).
- Silently invent a situational pattern (RBAC, audit, plugins, streaming, etc.) — STOP and ask.
- Write production-hardening reviews in Extension D (notes are fine, blockers are not).
- Track time-box compliance (the human facilitator owns that).
- Apply blanket TDD to auxiliary code (Extension G scopes it).
- Re-derive aesthetics per component (Extension H commits once).
- Invoke `superpowers:finishing-a-development-branch` (Extension F replaces it).
- Retry a failed Vercel deploy silently (Extension I writes a failure note and stops).
- Push environment variable values to Vercel during Extension I (detect and report only).
- Configure custom domains during Extension I (out of scope; default `*.vercel.app` URL only).
- Run `vercel postgres create` on the participant's behalf (note as a manual next step only).
