# Development Claude Skills

A curated repository of Claude Code Agent Skills for application development teams.
Skills automate code generation, scaffolding, and testing patterns for common application
frameworks using Claude Code as an AI pair-programmer.

## Adding This Repository to Your Project

Clone once, then add to any project:

```bash
git clone git@github.com:kavinksm/development-claude-skills.git ~/.claude/skill-repos/development
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
| `node-frontend` | `/node-frontend [area] [feature-name]` | "react component", "react hook", "useQuery", "zustand", "react router", "react hook form", "zod validation", "vitest", "playwright", "tailwind", "vite config", "frontend", "SPA" | Components & Hooks, State Management, Routing, Forms, API Client, Testing, Styling, Build & Deploy, Project Scaffold |

## Repository Layout

```
CLAUDE.md                              # repo overview (you are here)
skills/<skill-name>/SKILL.md           # skill instructions (frontmatter + body)
skills/<skill-name>/references/        # code reference templates loaded on demand
```

## Contribution Guidelines

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

### Code Template Standards (for reference files)

- All configurable values must use placeholder variables — never hardcode names
- Comment every non-obvious pattern with a brief explanation
- Use the latest stable framework patterns (Spring Boot 3.x / React 19 / Vite 6)
- Every code pattern must show matching test examples where applicable
- **Spring Boot**: prefer constructor injection; follow controller → service → repository layering
- **Node Frontend**: use TypeScript strict mode; functional components only; TanStack Query for server state
