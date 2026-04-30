<p>
  <a href="https://www.eso.org">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="https://www.eso.org/public/archives/logos/screen/eso-logo-p60b.jpg">
      <source media="(prefers-color-scheme: light)" srcset="https://www.eso.org/public/archives/logos/screen/eso-logo-p60b.jpg">
      <img alt="ESO" src="https://www.eso.org/public/archives/logos/screen/eso-logo-p60b.jpg" width="200">
    </picture>
  </a>
</p>

# ESO AI Skills

A collection of AI agent skills for astronomers working with ESO data services and tools.

These skills are designed to be small, composable, and easy to adapt. They work with any AI coding agent — GitHub Copilot, Claude Code, Cursor, and others. Each skill encodes expert knowledge about a specific ESO service or workflow so you don't have to repeat yourself.

> This repository may eventually move to the ESO GitHub organisation.

## Quickstart

Clone the repository and symlink the skills you want into your agent's skills directory:

```bash
git clone https://github.com/szampier/skills.git eso-skills
```

**GitHub Copilot CLI:**
```bash
ln -s $(pwd)/eso-skills/skills/archive/eso-tap-obs ~/.agents/skills/user/eso-tap-obs
```

**Claude Code:**
```bash
ln -s $(pwd)/eso-skills/skills/archive/eso-tap-obs ~/.claude/skills/eso-tap-obs
```

Then invoke the skill in your agent session (e.g. `/eso-tap-obs`).

## Skills

### Archive

Skills for querying and working with the [ESO Science Archive](https://archive.eso.org).

- **[eso-tap-obs](./skills/archive/eso-tap-obs/SKILL.md)** — Query `ivoa.ObsCore` via the TAP protocol. Translates natural-language requests into ADQL queries, TAP sync URLs, and ESO Science Portal links. Resolves target names via SIMBAD.

## Contributing

1. Pick a category under `skills/` (or propose a new one)
2. Create `skills/<category>/<skill-name>/SKILL.md` following the template below
3. Add an entry to the category's `README.md` and the top-level `README.md`
4. Open a pull request

### Skill template

```markdown
---
name: skill-name
description: One sentence. Use when [specific triggers].
---

# Skill Name

## Quick start

[Minimal working example]

## Workflows

[Step-by-step process]
```

The `description` field is what the agent reads to decide whether to load the skill — make it specific.
