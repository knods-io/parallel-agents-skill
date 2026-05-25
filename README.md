# Parallel Agents Skill

A coordination pattern for shipping multiple independent features in one session using parallel AI agent workers, hard isolation, and a human-gated deployment flow.

> **Trust your agents enough to parallelize. Distrust them enough to gate.**

Read the full rationale: [Trust your agents enough to parallelize. Distrust them enough to gate.](https://www.linkedin.com/pulse/trust-your-agents-enough-parallelize-distrust-them-gate-rolf-ruiz-y3wvc/)

## What this is

This is **not** a library or a package you install. It's a **skill definition** — a structured prompt that teaches your AI coding assistant how to coordinate parallel work safely.

The core idea:

1. You ask for 2+ independent features in one message
2. The assistant spawns parallel workers (Atlas, Argus, Janus...) in isolated git worktrees
3. A validator (Themis) merges and checks the combined result
4. A smoke tester (Apollo) verifies things actually work at runtime
5. The result lands on `staging` — never on `main`
6. **You** decide when to ship to production

```
                                ┌─→ Atlas   (worktree + branch worker/atlas-*)
[user request]  spawn parallel ─┼─→ Argus   (worktree + branch worker/argus-*)
                                └─→ Janus   (worktree + branch worker/janus-*)
                                       │
                                       ▼
                                   Themis  (merge + validate)
                                       │
                                       ▼
                                   Apollo  (smoke test)
                                       │
                                       ▼
                              staging ← (you test here)
                                       │
                                       ▼
                              "ship it" → main
```

## How to use this

**Do not copy-paste this skill verbatim into your project.** Your stack, your branching model, your test tooling, and your deployment flow are different from the project this was born in. Instead:

### 1. Read the skill

Read [`SKILL.md`](SKILL.md) end to end. Understand the flow, the roles, the anti-patterns, and the philosophy.

### 2. Ask your LLM to adapt it

Open a conversation with your AI coding assistant and ask it to implement this pattern for **your** project. For example:

> "Read this skill definition and adapt it to our project. We use [your stack]. Our branching model is [your branches]. Our tests run with [your test runner]. Our deploy target is [your deploy]. Keep the mythological worker names and the core flow (parallel workers → Themis validation → Apollo smoke test → staging → human gate → main), but tailor every detail to how we actually work."

### 3. What to customize

| Aspect | What to tailor |
|---|---|
| **Worker roles** | Atlas/Argus/Janus are generic starting points. Rename or reassign based on your codebase domains (e.g. "Atlas = database migrations", "Argus = API endpoints") |
| **Validation step** | Replace `npx tsc --noEmit` with your linter, type checker, or build command |
| **Smoke testing** | Replace Chrome DevTools MCP + curl with your test harness (Playwright, Cypress, pytest, etc.) |
| **Branch model** | The skill assumes `main` + `staging`. Adapt if you use `develop`, `release/*`, or trunk-based development |
| **Deployment** | The skill pushes to `staging` via git. Adapt if you deploy via CI/CD pipelines, Docker, Vercel, etc. |
| **Agent spawning** | The skill uses Claude Code's `Agent` tool with `isolation: "worktree"`. Adapt the spawning mechanism to your LLM's capabilities |

### 4. Install it

Where you place the adapted skill depends on your tool:

**Claude Code**
```
.claude/skills/parallel-delivery/SKILL.md
```

**Codex / OpenAI**
```
Reference the skill in your AGENTS.md or system prompt
```

**Cursor / Windsurf / other IDE agents**
```
Add to your project rules or custom instructions
```

**OpenClaw / Hermes / other agent frameworks**
```
Include in the agent's system prompt or skill registry
```

The skill is plain Markdown with YAML frontmatter. Any LLM that reads Markdown can use it.

## The roster

| Role | Name | Domain |
|---|---|---|
| Implementer | **Atlas** | Structural work, heavy lifting |
| Implementer | **Argus** | Cross-file bug fixes, edge cases |
| Implementer | **Janus** | Migrations, transitions |
| Implementer | **Hermes** | Network, API plumbing (backup) |
| Implementer | **Hephaestus** | Build tooling, infra (backup) |
| Validator | **Themis** | Integration merge, type checks, build sanity |
| Smoke tester | **Apollo** | Runtime verification, live tests |

Scale the implementer count to match your batch size. Themis and Apollo always run.

## How this differs from existing tools

There are excellent tools for running AI agents in parallel. This skill covers a different layer.

| Tool | What it does | What it doesn't do |
|---|---|---|
| [parallel-code](https://github.com/johannesjo/parallel-code) | Electron app that runs Claude/Codex/Gemini side-by-side in worktrees | No integration validation, no smoke testing, no staging gate |
| [claude-parallel-agents](https://github.com/sean-rowe/claude-parallel-agents) | Launches multiple Claude Code instances with state tracking and auto-restart | No merge validation, no runtime testing, no human-gated promotion |
| Claude Code `/batch` | Built-in command that spawns worktree-isolated agents for parallel migrations | Each agent creates its own PR — no integration branch, no unified validation |
| **This skill** | The full delivery ceremony after the parallel work is done | Not a runtime or launcher — it's the pipeline definition |

The existing tools solve **"how do I run multiple agents at once without file conflicts?"** This skill solves **"how do I safely merge, validate, test, and ship what those agents produced — with a human as the final gate?"**

The pipeline: parallel workers → integration merge (`--no-ff`) → type/build validation (Themis) → runtime smoke tests (Apollo) → staging promotion → human authorization → production merge, with feature-level rollback via `git revert -m 1`. That's the part nobody else has published as a portable, LLM-agnostic skill.

## Key principles

- **Isolation is non-negotiable.** Workers must not share a working directory. Git worktrees, separate clones, or containers — pick one, but don't skip it.
- **The integration branch is the validation buffer.** Worker branches never merge directly into staging. The integration branch is where bad merges go to die quietly.
- **`--no-ff` merges are the revert surface.** One merge commit per feature means one `git revert` removes one complete feature, not a tangle of individual commits.
- **`main` is sacred.** The flow never touches `main` until the human says so. No exceptions.
- **Don't manufacture ambiguity.** If the assistant can proceed with reasonable assumptions, it should state them and go. Questions are for real decisions, not safety theater.
- **Type-clean is not runtime-correct.** Themis checks types. Apollo checks behavior. Both must pass.

## License

MIT

## Author

[Rolf Ruiz](https://linkedin.com/in/rolfruiz) — Creative Technology
