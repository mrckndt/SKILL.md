# SKILLS.md

A collection of Agent Skills, following the [Agent Skills specification](https://agentskills.io/specification).

## Layout

```
skills/
└── <skill-name>/
    ├── SKILL.md      # Required: YAML frontmatter + instructions
    ├── scripts/      # Optional: executable code
    ├── references/   # Optional: documentation
    └── assets/       # Optional: templates, resources
.opencode/
└── commands/
    └── <skill-name>.md
```

Each skill is a directory whose name matches the `name` field in its `SKILL.md` frontmatter. See the specification for required fields (`name`, `description`) and optional ones (`license`, `compatibility`, `metadata`, `allowed-tools`).

## Use

`/token-saver` starts token saver. Arguments: `on` or `start` start it, `off` or `stop` stop it, `status` reports the mode. Plain `token-saver on`, `token-saver start`, `token-saver off`, `token-saver stop`, "stop token saver", and "turn off token saver" also work. A bare "token saver" does not start it.

`/intent-review` checks agent-generated work against what you asked for ("intent"), not only for bugs. `plan` reviews a plan before code exists, `review` (the default when there are changes) reviews uncommitted changes, a branch, a commit range, a path, or a PR, and `explain` walks through how code runs in plain language. `high` raises review effort, `--explain` adds a walkthrough to a review, `--save` stores a plan contract in `.intent-review/`, and `--ask-model` lets you pick the reviewer model. The review stages follow [Maddila et al., 2026](https://arxiv.org/abs/2607.29516v2) (Intent, Drift, Spotlight).

## Validate

```bash
skills-ref validate ./skills/<skill-name>
```
