# Operator Development Skill

Reusable guidance for building, reviewing, and refactoring Kubernetes operators and controllers whose reconciliation is deterministic, composable, and recoverable.

## Quick install

```sh
npx skills add stefanm8/operator-development-skill
```

## What it covers

- Desired-state functions built by factories, served by thin adapters, and composed from child constructors and small interfaces.
- Controller shapes: composite, decorator, external-resource sync, and one-shot action, with mappings onto reconciler frameworks, informer loops, hook-based controllers, and event-handler frameworks.
- Reconciliation invariants: failures that never shrink the desired set, deterministic output, stable conditions, concurrency, and confirmed cleanup.
- CRD and status design: schemas, validation, conditions, deterministic names, and label and annotation contracts.
- Proportionate testing: whole-object constructor tests, desired-state tests with fakes, adapter invariants, and high-value scenarios.

Existing project architecture, runtime, and conventions take precedence.

## Repository structure

```text
.
├── README.md
├── SKILL.md
└── references/
    ├── api-design.md
    ├── architecture.md
    ├── controller-shapes.md
    ├── reconciliation.md
    └── verification.md
```

Start with [SKILL.md](SKILL.md). Load supporting references only when relevant:

| Reference | Use for |
| --- | --- |
| [Controller shapes](references/controller-shapes.md) | Per-shape rules, ownership limits, related objects, and runtime mappings |
| [Reconciliation](references/reconciliation.md) | Failure handling, outcomes, concurrency, external systems, cleanup, and readiness |
| [API design](references/api-design.md) | CRD changes, schemas, spec and status, conditions, names, labels, and versions |
| [Architecture](references/architecture.md) | Layout, adapters, wiring, process hygiene, naming, and code conventions |
| [Verification](references/verification.md) | Test selection, fakes, runtime-specific notes, invariant tests, and scenarios |

## Manual installation

To install the skill for Codex, clone the repository into your skills directory.

```sh
mkdir -p ~/.codex/skills
git clone https://github.com/stefanm8/operator-development-skill.git ~/.codex/skills/operator-development
```

For Claude Code, use `~/.claude/skills/operator-development` as the clone destination. If you already have a clone, update it with the command below.

## Usage

```text
Use $operator-development to add a controller that provisions an external team for each Team resource.
```

```text
Use $operator-development to review this reconciler's cleanup and status handling.
```

The skill directs the agent to inspect the project's runtime contract first, preserve established conventions, and choose verification appropriate to the change.

## Updating a cloned installation

```sh
git -C ~/.codex/skills/operator-development pull --ff-only
```

For Claude Code, use the path to your Claude Code installation of the skill.
