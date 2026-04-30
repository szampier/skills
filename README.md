# skills

A collection of AI agent skills for [GitHub Copilot CLI](https://githubnext.com/projects/copilot-cli) and other coding agents.

## What are skills?

Skills are modular instruction sets that extend AI coding agents with specialized knowledge and workflows. Each skill is a directory containing a `SKILL.md` file (and optionally reference files and scripts).

## Structure

```
skills/
└── skill-name/
    ├── SKILL.md        # Main instructions (required)
    ├── REFERENCE.md    # Detailed docs (optional)
    ├── EXAMPLES.md     # Usage examples (optional)
    └── scripts/        # Utility scripts (optional)
```

## Installing a skill

### GitHub Copilot CLI

```bash
gh copilot skill install szampier/skills/<skill-name>
```

Or clone the repo and symlink a skill into your local skills directory:

```bash
git clone https://github.com/szampier/skills.git
ln -s $(pwd)/skills/<skill-name> ~/.agents/skills/user/<skill-name>
```

## Skills

| Skill | Description |
|-------|-------------|
| *(more coming soon)* | |

## Contributing

1. Create a new directory with a kebab-case name
2. Add a `SKILL.md` following the [template](#skill-template)
3. Open a pull request

### Skill template

```markdown
---
name: skill-name
description: One-line description. Use when [specific triggers].
---

# Skill Name

## Quick start

[Minimal working example]

## Workflows

[Step-by-step processes]
```

The `description` field is what the agent reads to decide whether to load the skill — make it specific and trigger-friendly.
