# SKILLS.md

A collection of Agent Skills, following the [Agent Skills specification](https://agentskills.io/specification).

## Layout

```
skills/
└── <skill-name>/
    ├── SKILL.md          # Required: YAML frontmatter + instructions
    ├── scripts/          # Optional: executable code
    ├── references/       # Optional: documentation
    └── assets/           # Optional: templates, resources
```

Each skill is a directory whose name matches the `name` field in its `SKILL.md` frontmatter. See the specification for required fields (`name`, `description`) and optional ones (`license`, `compatibility`, `metadata`, `allowed-tools`).

## Use

`opencode.json` registers the `skills/` directory and a `/token-saver` command. Open OpenCode at the repo root so the relative `skills` path resolves.

`/token-saver` starts token saver. Arguments: `on` or `start` start it, `off` or `stop` stop it, `status` reports the mode. Plain `token-saver on`, `token-saver start`, `token-saver off`, `token-saver stop`, "stop token saver", and "turn off token saver" also work. A bare "token saver" does not start it.

## Validate

```bash
skills-ref validate ./skills/<skill-name>
```
