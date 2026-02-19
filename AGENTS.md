# Skills Maintenance

Guidelines for creating and updating skills in this repository.

## Repository Structure

```
skills/
├── {skill-name}/
│   ├── SKILL.md            # Index: preferences, quick reference, reference table
│   └── references/
│       └── *.md            # Individual reference files (loaded on demand)
```

## Skill Format

### SKILL.md

Index file with YAML frontmatter. Loaded when the skill triggers.

```markdown
---
name: {kebab-case-name}
description: {what it does + when to use it}
license: MIT
metadata:
  author: marsidev
  version: "YYYY.MM.DD"
---
```

Required frontmatter: `name` (string), `description` (string).

The description is the primary triggering mechanism. Include both what the skill does AND specific triggers/contexts for when to use it.

### Reference Files

One concept per file. Loaded on demand, not all at once.

Each reference should have frontmatter:

```markdown
---
name: {reference-name}
description: {brief description}
---
```

## Writing Guidelines

1. **Write for AI agents, not humans** — synthesize for LLM consumption, don't copy docs verbatim
2. **Be concise** — the context window is a shared resource. Challenge each paragraph: does the agent really need this?
3. **Code over prose** — prefer examples over explanations. Show the pattern, not the theory
4. **One concept per reference file** — keeps loading granular
5. **Progressive disclosure** — SKILL.md stays lean (~100 lines), references loaded as needed
6. **Skip common knowledge** — don't explain what `ref()` does. Focus on conventions, gotchas, and patterns that differ from what LLMs already know
7. **Prefer imperative form** — "Use X" not "You should use X"
8. **No promotional language** — direct, factual, technical

## Adding a New Skill

1. Create `skills/{name}/SKILL.md` with required frontmatter
2. Create `skills/{name}/references/` with reference files
3. Ensure SKILL.md links to all references with descriptions
4. Update root `README.md` skills table

## Updating a Skill

1. Update affected reference files
2. Update `SKILL.md` version date in frontmatter
3. Update reference table in SKILL.md if references were added/removed
