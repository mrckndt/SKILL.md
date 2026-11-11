# SKILLS.md

Personal collection of Agent Skills, following the [Agent Skills specification](https://agentskills.io/specification).

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

## Validate

```bash
skills-ref validate ./skills/<skill-name>
```
