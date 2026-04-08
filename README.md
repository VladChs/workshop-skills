# Workshop Skills

Skills for the **Build Your Idea** workshop: ship any idea in 4 hours with Claude Code as your co-pilot.

## What's inside

- **`workshop-pipeline`** — the entry-point skill. Runs the full superpowers pipeline (brainstorm → plan → execute → review → finish) with workshop-tuned extensions for product-design discipline, pragmatic testing, and frontend aesthetics.
- **`workshop-architecture`** — a reference skill loaded by the pipeline at plan-writing time. 25 universal architecture patterns that shape the structure of every workshop build.

## Pre-reqs

- Claude Code installed and authenticated
- `superpowers` plugin installed (the workshop pipeline wraps it):

  ```
  /plugin marketplace add anthropics/claude-plugins-official
  /plugin install superpowers@claude-plugins-official
  ```

- For frontend projects, also install `frontend-design`:

  ```
  /plugin install frontend-design@claude-plugins-official
  ```

## Install

```
/plugin marketplace add VladChs/workshop-skills
/plugin install workshop@workshop-marketplace
```

## Use

Start a fresh Claude Code session in your workshop project and say:

> "Start the workshop build."

Claude will invoke `workshop-pipeline` and walk you through the phases.

## Update

```
/plugin update workshop@workshop-marketplace
```
