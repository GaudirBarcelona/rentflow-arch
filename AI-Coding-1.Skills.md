# Proposed skills/plugins

## Installation

## [caveman](https://github.com/juliusbrussee/caveman) output optimization

Caveman reduces verbose agent output while preserving code, commands, paths,
errors, numbers, and safety information.

Install globally:

```bash
npx skills add JuliusBrussee/caveman -g
```

Choose:
- caveman: reduce LLM explanations
- caveman-explore: Explore alternative designs
- caveman-optimize: Optimize afterwards
- caveman-review: Review implementation
- investigate-first: Deep investigation before coding

### [Grill-with-docs](https://github.com/mattpocock/skills)

Install the skill from the `mattpocock/skills` repository:

```bash
npx skills add https://github.com/mattpocock/skills --skill grill-with-docs -g
```

Restart the Copilot session, verify that the skill is available, and invoke it with `/grill-with-docs` from the repository you want it to document.

### [Elephant-Goldfish Model](https://github.com/google/elephant-goldfish-model)

Register the EGM marketplace and install its plugin in GitHub Copilot CLI:

```bash
copilot plugin marketplace add google/elephant-goldfish-model
copilot plugin install egm@egm
```

Start a new Copilot CLI session and verify the installed skills or commands before running `egm-brainstorm`.

### [OpenSpec](https://github.com/Fission-AI/openspec)

Install the standalone OpenSpec CLI, then initialize it in the target repository:

```bash
curl -fsSL https://openspec.dev/cli/install.sh | sh
openspec init
```

### [Superpowers](https://github.com/obra/superpowers)

The upstream `obra/superpowers` repository does not currently provide an official GitHub Copilot CLI package. For Copilot CLI, use the community adapter only after your organization reviews it:

```bash
copilot plugin marketplace add DwainTR/superpowers-copilot
copilot plugin install superpowers@superpowers-copilot
```

Start a new session and run `/skills list`. Confirm that the required Superpowers skills are visible before using the workflow.

### [shadcn-improve](https://github.com/shadcn/improve)

Audits any codebase and writes implementation plans for other agents to execute.
```bash
npx skills add shadcn/improve -g
```