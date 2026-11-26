# SKILLS.md

A collection of Agent Skills, following the [Agent Skills specification](https://agentskills.io/specification).

## Layout

```
skills/
├── <skill-name>/
│   ├── SKILL.md      # Required: YAML frontmatter + instructions
│   ├── scripts/      # Optional: executable code
│   ├── references/   # Optional: documentation
│   └── assets/       # Optional: templates, resources
└── compound-engineering-md/
    └── <ce-skill>/
        └── SKILL.md  # Grouped Markdown port of Compound Engineering skills
.opencode/
└── commands/
    └── <skill-name>.md
```

Each skill is a directory whose name matches the `name` field in its `SKILL.md` frontmatter. See the specification for required fields (`name`, `description`) and optional ones (`license`, `compatibility`, `metadata`, `allowed-tools`); skills may also include host-specific frontmatter fields (`argument-hint`, `disable-model-invocation`).

## Use

`/token-saver` starts token saver. Arguments: `on` or `start` start it, `off` or `stop` stop it, `status` reports the mode. Plain `token-saver on`, `token-saver start`, `token-saver off`, `token-saver stop`, "stop token saver", and "turn off token saver" also work. A bare "token saver" does not start it.

`/intent-review` checks agent-generated work against your intent (`plan`, `review`, `explain`) following [Maddila et al., 2026](https://arxiv.org/abs/2607.29516v2) (Intent, Drift, Spotlight). Supports `high`, `--explain`, `--save`, and `--ask-model`.

`skills/compound-engineering-md/` bundles portable, script-free Markdown ports of the Compound Engineering skills (`ce-brainstorm`, `ce-plan`, `ce-work`, `ce-code-review`, `ce-compound`, etc.). See [its README](skills/compound-engineering-md/README.md) for the full list and workflow.

## Validate

```bash
skills-ref validate ./skills/<skill-name>
```
