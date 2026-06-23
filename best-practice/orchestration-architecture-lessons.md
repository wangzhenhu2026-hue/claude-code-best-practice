# Orchestration Architecture Lessons

Architecture lessons distilled from the `weather-orchestrator` example workflow — transferable patterns for building reliable Claude Code orchestrations.

<table width="100%">
<tr>
<td><a href="../">← Back to Claude Code Best Practice</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

---

The weather system (see [orchestration-workflow.md](../orchestration-workflow/orchestration-workflow.md)) looks trivial — fetch Dubai's temperature, draw a card. But it is deliberately over-decomposed into a **Command → Agent → Skill** chain to teach a single idea:

> **Replace "trust the LLM to behave" with "make correctness structural."**

The more complex and production-bound a task is, the more these lessons pay off.

## 1. Separation of Concerns — one job per layer

| Layer | Sole responsibility | Must NOT touch |
|-------|--------------------|----------------|
| Command (`weather-orchestrator`) | Orchestrate + talk to the user | Fetching data, drawing output |
| Agent (`weather-agent`) | Fetch data | Drawing, writing files |
| Skill (`weather-fetcher`) | Hit the network for raw data | Transforming, writing files |
| Skill (`weather-svg-creator`) | Render output | Re-fetching data |

When one agent or prompt fetches *and* computes *and* writes files *and* chats with the user, it will cut a corner somewhere. Split **decide / fetch / render** so each piece is independently testable, replaceable, and reusable.

## 2. Tool allowlists turn convention into hard constraints

The sharpest move in the example is enforcing architecture via **permissions**, not prose:

- `weather-agent` → `allowedTools: [Read, Skill]` — deliberately **no** network tools
- `weather-fetcher` → `allowed-tools: ["WebFetch(*)"]` — the **only** point in the system that can reach the network

The agent definition even says: *"Your tool allowlist intentionally excludes network tools — if you find yourself needing one, that is a signal you are bypassing the skill."*

Don't *ask* the model to follow rules in a prompt (it will take shortcuts). Use **permission boundaries** so the violation is impossible. Converge high-risk capabilities (network, file writes, execution) onto a single responsible point — least privilege for agents.

## 3. Fail-closed guardrails — stop on error, never fabricate

```
Fail-closed guardrail: If the agent does not return a numeric
temperature and unit, DO NOT proceed. Report the failure and stop.
```

In a multi-step chain the real danger is not that a step fails — it's that a **downstream step runs with dirty data** (an agent fails to fetch the temperature, but the SVG still renders `undefined°F`). A "stop unless you have valid input" check at every handoff converts a silent bad output into a clear, locatable halt.

## 4. Allocate models/resources per component, not chain-wide

- Command → `model: haiku` (cheap + fast, orchestration only)
- Agent → `model: sonnet` (fetching + reasoning, worth a stronger model)
- Agent also carries `maxTurns`, `memory: project`, and scoped hooks

Don't run the whole chain on one top-tier model. **Cheap model for orchestration, strong model for the reasoning step** — a key cost/quality lever.

## 5. (Cautionary) Budget resources to match flow complexity

The example originally shipped with `maxTurns: 5` on `weather-agent`. The real flow is: invoke skill → WebFetch → receive a large JSON blob → parse → report. Five turns were exhausted *before* the final report, so the agent returned a mid-process line instead of the temperature — and the fail-closed guardrail correctly blocked the render step.

When setting hard caps like `maxTurns`, estimate from the agent's **worst-case real step count**, not a guess. Too loose wastes tokens; too tight truncates the agent right before it reports — a failure mode that hides as a half-finished sentence.

> **Debugging tip:** if a subagent's returned message looks truncated, check whether `tool_uses` exactly equals `maxTurns`. That equality is the signature of a turn-budget truncation.

## 6. Progressive disclosure — keep `SKILL.md` lean

`weather-svg-creator` keeps its `SKILL.md` short and pushes the SVG template, color specs, and examples into `reference.md` / `examples.md`, read only when needed.

Don't stuff every detail into a file that gets loaded into context. **The main file says *when* and *what*; details live in companion files loaded on demand** — saving context and easing maintenance.

## Summary

| Lesson | Mechanism in the example |
|--------|--------------------------|
| Separation of concerns | Command / Agent / two Skills, each single-purpose |
| Structural permissions | `allowedTools` converges network access to one skill |
| Fail-closed | "Stop unless you have a numeric temperature + unit" |
| Per-component resources | haiku command + sonnet agent + scoped `maxTurns`/`memory` |
| Resource budgeting | `maxTurns` must fit worst-case step count |
| Progressive disclosure | Lean `SKILL.md` + companion `reference.md`/`examples.md` |

The toy task is the wrapper; the lesson is the architecture.
