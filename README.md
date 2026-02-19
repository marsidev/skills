# marsidev's Skills

Opinionated [Agent Skills](https://skills.sh) for web development. Conventions, patterns, and gotchas distilled from official docs and production codebases.

## Installation

```bash
# Install a specific skill
npx skills add marsidev/skills --skill vue

# Install all skills
npx skills add marsidev/skills --skill='*'

# Install globally (available across all projects)
npx skills add marsidev/skills --skill='*' -g
```

Learn more about the CLI at [vercel-labs/skills](https://github.com/vercel-labs/skills).

## Skills

| Skill | Description |
|-------|-------------|
| [vue](skills/vue) | Vue 3 Composition API conventions, script setup macros, reactivity, and component patterns |

## Sources

Skills are hand-written, informed by:

- [Vue.js official documentation](https://vuejs.org)
- [Nuxt official documentation](https://nuxt.com)
- [Anthony Fu's skills](https://github.com/antfu/skills) (API reference generation)
- [vuejs-ai/skills](https://github.com/vuejs-ai/skills) (gotchas and best practices)
- Personal conventions from production Vue/Nuxt codebases

## License

[MIT](LICENSE)
