# Development Claude Skills

A curated repository of Claude Code Agent Skills for application development teams.
Skills automate code generation, scaffolding, and testing patterns for common application
frameworks using Claude Code as an AI pair-programmer.

## Quick Start

Clone once, then add to any project:

```bash
git clone git@github.com:kavinksm/development-skills.git ~/.claude/skill-repos/development
```

Add to a specific project:

```bash
claude --add-dir ~/.claude/skill-repos/development
```

Or add globally so all your projects can use the skills:

```bash
claude --add-dir ~/.claude/skill-repos/development --global
```

After adding, skills are available immediately — no restart needed. Type `/springboot`
or describe what you want (e.g., "create a Spring Boot REST controller with validation")
and Claude will invoke the relevant skill automatically.

## Available Skills

| Skill | Invoke | Auto-trigger phrases | Covers |
|---|---|---|---|
| `springboot` | `/springboot [area] [feature-name]` | "spring boot", "spring mvc", "rest controller", "jpa entity", "spring security", "junit test", "mockito", "@SpringBootTest" | Web layer, Data access, Security, Testing, Messaging, Caching, Actuator, Configuration, Build & Deploy |

## Repository Layout

```
README.md                              # this file
CLAUDE.md                              # repo overview and contribution guidelines
skills/<skill-name>/SKILL.md           # skill instructions (frontmatter + body)
skills/<skill-name>/references/        # code reference templates loaded on demand
```

## Contributing

### Adding a New Skill

1. Create `skills/<skill-name>/` directory — name must be kebab-case
2. Create `SKILL.md` with required frontmatter fields:
   - `name`: matches directory name exactly (lowercase, hyphens, no consecutive hyphens)
   - `description`: one paragraph, keyword-rich for auto-invocation matching
   - `allowed-tools`: space-separated list (`Read Grep Glob Write Edit Bash`)
   - `argument-hint` (optional): short usage shown in autocomplete
3. Keep `SKILL.md` under 400 lines — move detailed reference content to `references/` files
4. Test locally: add this repo with `--add-dir`, then invoke the skill with `/skill-name`
5. Open a PR with before/after output examples in the description

### Code Template Standards

- All configurable values must use placeholder variables — never hardcode names
- Use the latest stable Spring Boot 3.x patterns (Jakarta EE, not javax)
- Every code pattern must show matching test examples where applicable
- Prefer constructor injection over field injection
- Follow clean architecture: controller → service → repository layering
