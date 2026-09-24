---
name: reactivity
description: shallowRef vs ref, and composable conventions
---

# Reactivity & Composables

## ref vs shallowRef

```ts
import { ref, shallowRef } from 'vue'

// ref — deep reactivity (tracks nested changes)
const user = ref({ name: 'John', profile: { age: 30 } })
user.value.profile.age = 31 // triggers reactivity

// shallowRef — only .value assignment triggers reactivity
const data = shallowRef({ items: [] })
data.value.items.push('new') // does NOT trigger reactivity
data.value = { items: ['new'] } // triggers reactivity
```

**Prefer `shallowRef`** for large data structures, external library instances, or when deep reactivity is unnecessary.

## Composables

### Naming

- Prefix with `use`: `useAuth`, `useMouse`, `useCounter`
- File matches function: `useAuth.ts` exports `useAuth`

### Structure

```ts
import { ref, readonly, onMounted, onUnmounted } from 'vue'

export function useMouse() {
  const x = ref(0)
  const y = ref(0)

  const update = (e: MouseEvent) => {
    x.value = e.pageX
    y.value = e.pageY
  }

  onMounted(() => window.addEventListener('mousemove', update))
  onUnmounted(() => window.removeEventListener('mousemove', update))

  return { x: readonly(x), y: readonly(y) }
}
```

### Key Rules

1. **Return plain objects with refs** — not `reactive()` (loses reactivity when destructured)
2. **Use `readonly()`** for state that consumers should not mutate
3. **Accept `MaybeRefOrGetter`** for flexible inputs (Vue 3.3+):

```ts
import { toValue, type MaybeRefOrGetter } from 'vue'

export function useFetch(url: MaybeRefOrGetter<string>) {
  watchEffect(async () => {
    const res = await fetch(toValue(url))
    data.value = await res.json()
  })
}

// All work:
useFetch('/api/users')
useFetch(urlRef)
useFetch(() => `/api/users/${props.id}`)
```

4. **Handle cleanup** — use `onUnmounted` or `onScopeDispose`
5. **No async composables** — they lose lifecycle context when awaited in other composables
6. **Top-level only** — never call inside event handlers, conditionals, or loops

<!--
Source references:
- https://vuejs.org/api/reactivity-core.html
- https://vuejs.org/guide/reusability/composables.html
-->
