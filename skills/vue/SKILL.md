---
name: vue
description: Vue 3 Composition API conventions, script setup macros, reactivity patterns, and component best practices. Use when writing .vue files, creating composables, working with defineProps/defineEmits/defineModel, or using Tailwind in Vue templates.
license: MIT
metadata:
  author: marsidev
  version: "2026.02.19"
---

# Vue 3 Development

> Based on Vue 3.5+. Always use Composition API with `<script setup lang="ts">`.

## Preferences

- Always `<script setup lang="ts">` — no Options API
- TypeScript for all component logic
- Prefer `shallowRef` over `ref` when deep reactivity is not needed
- Prefer Tailwind utility classes over custom CSS
- Avoid reactive props destructure — use `const props = defineProps<Props>()`
- Function declarations (`function handleClick()`) over arrow functions for component methods
- Early returns over if/else blocks

## Quick Template

```vue
<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'

const props = defineProps<{
  title: string
  count?: number
}>()

const emit = defineEmits<{
  update: [value: string]
}>()

const model = defineModel<string>()

const doubled = computed(() => (props.count ?? 0) * 2)

onMounted(() => {
  // ...
})
</script>

<template>
  <div>{{ title }} - {{ doubled }}</div>
</template>
```

## References

Load only the file relevant to your current task:

### Conventions

| Topic | Description | Reference |
|-------|-------------|-----------|
| Code Organization | Script setup ordering, template rules, styling, naming conventions | [conventions](references/conventions.md) |

### API

| Topic | Description | Reference |
|-------|-------------|-----------|
| Script Setup Macros | defineProps, defineEmits, defineModel, defineExpose, defineOptions, defineSlots, generics | [macros](references/macros.md) |
| Reactivity & Lifecycle | ref, shallowRef, computed, watch, watchEffect, composable patterns, effect scope | [reactivity](references/reactivity.md) |

### Patterns

| Topic | Description | Reference |
|-------|-------------|-----------|
| Component Patterns | Async emit callbacks, v-model patterns, slot usage, loading states | [patterns](references/patterns.md) |

### Gotchas

| Topic | Description | Reference |
|-------|-------------|-----------|
| Common Pitfalls | Reactivity traps, computed mistakes, watcher timing, TypeScript, Tailwind, performance | [gotchas](references/gotchas.md) |

## When to Load

- Editing a `.vue` component → [conventions](references/conventions.md) + [macros](references/macros.md)
- Writing a composable → [reactivity](references/reactivity.md)
- Implementing parent-child communication → [patterns](references/patterns.md)
- Something isn't working as expected → [gotchas](references/gotchas.md)

**Do not load all references at once.**
