# AI Coding Plugins and Skills Playbook

> A practical guide for choosing and combining specification, planning, implementation, architecture, visualization, and token-efficiency skills in AI-assisted software development.

## Recommended default workflow

For most non-trivial changes, use one primary workflow owner and add specialist tools only when they produce a distinct artifact.

```text
Understand existing code (optional)
  → Clarify the problem
  → Write/approve a specification
  → Plan implementation
  → Implement with tests
  → Review
  → Document or visualize
  → Retrospective / archive
```

A good default stack is:

1. **Understand Anything** for unfamiliar or large repositories.
2. **OpenSpec** *or* **mattpocock/skills** for requirements and durable design decisions.
3. **Superpowers** for disciplined implementation with worktrees, TDD, and reviews.
4. **shadcn/improve** for codebase audits and executor-ready remediation plans.
5. **Archify** for architecture communication and interactive diagrams.
6. **Caveman** only as an optional efficiency layer, not as a development methodology.

Understand Anything = understand the repository
grill-with-docs = establish vocabulary and decisions
OpenSpec = define the approved change
Superpowers = implement and verify
Archify = communicate architecture
Chronicle = learn from work across sessions
Caveman = reduce token consumption

> **Important:** Do not run several end-to-end methodologies for the same change. OpenSpec, mattpocock/skills, and Superpowers overlap in discovery and planning. Choose which one owns each artifact to avoid duplicate specifications, conflicting task lists, and documentation drift.

---

## Decision matrix

| Tool | Primary purpose | Best time to use | Main output | Modifies product code? | Best fit |
|---|---|---|---|---|---|
| [OpenSpec](https://github.com/Fission-AI/openspec) | Spec-driven change management | Before implementing a feature or behavioral change | Proposal, requirements, scenarios, design, tasks, archived change | During apply | Teams needing reviewable, versioned specifications |
| [mattpocock/skills](https://github.com/mattpocock/skills) | Composable engineering workflow and shared vocabulary | From discovery through review and retrospective | `GLOSSARY.md`, ADRs, specs, tickets, reviews | Yes, through implementation skills | Engineers who want modular skills and durable context |
| [Superpowers](https://github.com/obra/superpowers) | Opinionated end-to-end engineering methodology | From initial idea through branch completion | Design, plan, tested commits, reviews | Yes | Autonomous but controlled implementation |
| [shadcn/improve](https://github.com/shadcn/improve) | Audit and implementation-plan generation | Before refactoring, hardening, or backlog creation | Prioritized findings and self-contained plans | No, except delegated execution in worktrees | Using a strong model for analysis and cheaper models for execution |
| [Caveman](https://github.com/juliusbrussee/caveman) | Reduce response and tool-output token usage | Across sessions when verbosity or token cost is a problem | Shorter responses and compressed inputs | No methodology-level changes | Long or expensive agent sessions |
| [Archify](https://github.com/tt-a1i/archify) | Interactive architecture and explainer diagrams | During discovery, design review, onboarding, or communication | Standalone interactive HTML | No | Explaining systems and tracing flows visually |
| [Understand Anything](https://github.com/Egonex-AI/Understand-Anything) | Repository and knowledge-base comprehension | Before changing an unfamiliar or large codebase | Knowledge graph, dashboard, domain view, guided tours | No product-code changes | Onboarding, impact analysis, and architecture discovery |

---

## Which tool should own each phase?

| Phase | Recommended owner | Alternatives / specialists |
|---|---|---|
| Repository discovery | **Understand Anything** | Archify for a presentation-orient