# marsidev/skills

Opinionated [Agent Skills](https://skills.sh) for software development. Hand-written from production experience.

## Skills

| Skill | Description |
|-------|-------------|
| [review-pr](skills/review-pr) | PR code review with structured findings, severity tiers, and GitHub inline comments |
| [vue](skills/vue) | Vue 3 Composition API conventions, script setup macros, reactivity, and component patterns |

## Installation

```bash
# Interactively select skills to install, the agent(s), the scope (project or global), and the method (symlink or copy)
npx skills add marsidev/skills
```

## More Examples

```bash
# List all skills
npx skills add marsidev/skills --list

# Install specific skills
npx skills add marsidev/skills --skill review-pr --skill vue

# Install all skills
npx skills add marsidev/skills --all

# Install to specific agents
npx skills add marsidev/skills --agent claude-code --agent opencode

# Install all skills to specific agents
npx skills add marsidev/skills --skill '*' -a claude-code

# Install globally (available in all projects)
npx skills add marsidev/skills --skill review-pr -g

# Non-interactive installation (CI/CD friendly)
npx skills add marsidev/skills --skill review-pr -g -a claude-code -y

# Install specific skills to all agents
npx skills add marsidev/skills --agent '*' --skill review-pr

# Remove interactively (select from installed skills)
npx skills remove

# Remove specific skill by name
npx skills remove review-pr

# Remove from global scope
npx skills remove --global review-pr

# Remove from specific agents only
npx skills remove --agent claude-code cursor review-pr
```

Learn more about the CLI at [vercel-labs/skills](https://github.com/vercel-labs/skills).

## License

[MIT](LICENSE)

## Support

<div align="left">
  <a href='https://ko-fi.com/K3K1D5PRI' target='_blank'><img height='36' style='border:0px;height:36px;' src='https://storage.ko-fi.com/cdn/kofi2.png?v=6' border='0' alt='Buy Me a Coffee at ko-fi.com' /></a>
</div>