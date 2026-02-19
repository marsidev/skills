---
name: conventions
description: Script setup ordering, template rules, styling approach, and naming conventions for Vue SFCs
---

# Vue Conventions

## Script Setup Ordering

Follow this order inside `<script setup lang="ts">`. Add blank lines between sections.

```vue
<script setup lang="ts">
// 1. Imports
import { ref, computed, onMounted, watch } from 'vue'
import type { SomeType } from '@/types'

// 2. Nuxt-specific (if applicable)
useHead({ title: 'Page Title' })
definePageMeta({ layout: 'default' })

// 3. Component-specific types
interface Props {
  title: string
}

// 4. defineProps
const props = defineProps<Props>()

// 5. defineEmits
const emit = defineEmits<{
  update: [value: string]
}>()

// 6. defineModel (if applicable)
const model = defineModel<string>()

// 7. Constants
const MAX_ITEMS = 10

// 8. Stores / composables
const authStore = useAuthStore()

// 9. Refs
const isLoading = ref(false)
const items = ref<Item[]>([])

// 10. Computed
const filteredItems = computed(() => items.value.filter(i => i.active))

// 11. Functions
function handleClick() {
  // ...
}

// 12. Lifecycle hooks
onMounted(() => {
  // ...
})

onUnmounted(() => {
  // ...
})

// 13. Watchers (always add a comment explaining purpose)
// Refetch data when filter changes
watch(filter, () => {
  fetchData()
})
</script>

<template>
  <!-- Template content -->
</template>
```

## Template Rules

1. Self-closing tags for components without children: `<MyComponent />`
2. **kebab-case** for event names in templates: `@update-value`
3. **PascalCase** for component names: `<MyComponent>`
4. Prefer `:class` binding with arrays/objects for conditional classes
5. Prefer `v-if`/`v-else` over ternary expressions in templates
6. Use `#header` shorthand for slots, not `v-slot:header`
7. Same-name shorthand (Vue 3.4+): `:count` instead of `:count="count"`

## Styling

1. Prefer **Tailwind utility classes** in the template
2. **Avoid `<style>` blocks** unless overriding third-party library styles
3. If custom CSS is needed, use `<style scoped>`
4. Never use inline styles — use Tailwind classes instead
5. Never construct Tailwind class names dynamically (e.g., `` `bg-${color}-500` ``). Use mapping objects with complete class names instead.

## Naming

| Element | Convention | Example |
|---------|-----------|---------|
| Components | PascalCase | `UserProfile.vue` |
| Composables | camelCase with `use` prefix | `useUserData.ts` |
| Refs/reactive | camelCase | `isLoading`, `userData` |
| Functions | camelCase, verb-first | `fetchData`, `handleSubmit` |
| Constants | SCREAMING_SNAKE_CASE | `MAX_RETRIES` |
| Props interface | `Props` | `defineProps<Props>()` |
| Event handlers | `handle` prefix | `handleSave`, `handleDelete` |
| Loading states | `is` prefix | `isLoading`, `isSaving` |

## File Organization

- **Components > 300 lines**: split into smaller components or extract logic to composables
- **Types shared across components**: move to a dedicated `types.ts` file
- **Constants shared across files**: move to `constants.ts`
