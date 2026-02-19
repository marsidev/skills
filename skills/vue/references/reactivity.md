---
name: reactivity
description: Vue 3 reactivity system, watchers, lifecycle hooks, effect scope, and composable patterns
---

# Reactivity, Lifecycle & Composables

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

## computed

```ts
// Read-only
const doubled = computed(() => count.value * 2)

// Writable
const plusOne = computed({
  get: () => count.value + 1,
  set: (val) => { count.value = val - 1 }
})
```

## reactive & readonly

```ts
const state = reactive({ count: 0, nested: { value: 1 } })
const readonlyState = readonly(state)
```

`reactive()` loses reactivity on destructuring. Use `ref()` or `toRefs()` instead.

## Watchers

### watch

```ts
// Single ref
watch(count, (newVal, oldVal) => {
  console.log(`Changed: ${oldVal} → ${newVal}`)
})

// Getter (for props or computed expressions)
watch(
  () => props.id,
  (id) => fetchData(id),
  { immediate: true }
)

// Multiple sources
watch([firstName, lastName], ([first, last]) => {
  fullName.value = `${first} ${last}`
})

// Depth limit (Vue 3.5+)
watch(state, callback, { deep: 2 })

// Fire once (Vue 3.4+)
watch(source, callback, { once: true })
```

### watchEffect

Runs immediately, auto-tracks dependencies:

```ts
watchEffect(async () => {
  const controller = new AbortController()

  // Cleanup on re-run or unmount (Vue 3.5+)
  onWatcherCleanup(() => controller.abort())

  const res = await fetch(`/api/${id.value}`, { signal: controller.signal })
  data.value = await res.json()
})
```

Pause/resume (Vue 3.5+):

```ts
const { pause, resume, stop } = watchEffect(() => {})
```

### Flush Timing

```ts
// 'pre' (default) — before component update
// 'post' — after component update (access updated DOM)
// 'sync' — immediate, use with caution

watch(source, callback, { flush: 'post' })
watchPostEffect(() => {}) // alias for flush: 'post'
```

## Lifecycle Hooks

```ts
import {
  onMounted,
  onUnmounted,
  onBeforeMount,
  onBeforeUnmount,
  onUpdated,
  onErrorCaptured,
  onActivated,     // KeepAlive
  onDeactivated,   // KeepAlive
} from 'vue'

onMounted(() => {
  // DOM is ready
})

onUnmounted(() => {
  // cleanup timers, listeners, subscriptions
})

// Error boundary
onErrorCaptured((err, instance, info) => {
  console.error(err)
  return false // stop propagation
})
```

## Effect Scope

Group reactive effects for batch disposal:

```ts
import { effectScope, onScopeDispose } from 'vue'

const scope = effectScope()

scope.run(() => {
  const count = ref(0)
  const doubled = computed(() => count.value * 2)
  watch(count, () => console.log(count.value))

  onScopeDispose(() => {
    console.log('scope disposed')
  })
})

scope.stop() // disposes all effects
```

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
- https://vuejs.org/api/composition-api-lifecycle.html
- https://vuejs.org/guide/reusability/composables.html
-->
