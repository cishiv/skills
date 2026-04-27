# Skill Systems

> Early personal-workflow experiment. No guarantee this works well outside my setup.

A **system** is a set of skills intended to be used together. Skills can still be installed and used individually, but there's an implicit dependency graph between some of them.

## Pattern

Each system is a **single bundled skill** that nests its sub-skills as folders, mirroring the [`product-team`](https://github.com/alirezarezvani/claude-skills/tree/main/product-team) layout:

```
{system-name}/
├── SKILL.md              # top-level dispatcher: routes to the right sub-skill
├── README.md             # human-readable overview (optional)
└── {sub-skill-name}/
    ├── SKILL.md
    ├── references/       # static references the sub-skill loads on demand
    ├── scripts/          # automation (Python, Bun, etc.) — optional
    └── assets/           # templates, fixtures — optional
```

The top-level `SKILL.md` doesn't do the work itself — it explains the pipeline and routes the user to the appropriate sub-skill.

Skills shared across multiple systems live at the repo root under `../skills/{shared-skill}/` and are referenced from each system's top-level `SKILL.md`.

## Systems

### [`mvp-builder/`](mvp-builder/)

Idea → deployed MVP, using the user's three template repos (`kitchen-sink-ts`, `kitchen-sink-twotier`, `statix`). Pipeline: `detailed-specification` → `mvp-specification` → `build-mvp` → `railway-deployment-{template}`, with `extend-features` as a sibling entry point for existing repos.

Currently implemented stages: `detailed-specification`. Others scoped on demand.

### Blog Synthesis Skill System

Planned. Translate raw notes into consumable blog posts with tone maintained.
