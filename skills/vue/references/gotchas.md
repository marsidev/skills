---
name: gotchas
description: Common Vue pitfalls — reactivity traps, computed mistakes, watcher timing, TypeScript, Tailwind dynamic classes, and performance
---

# Common Gotchas

## Reactivity

### Destructuring reactive() loses reactivity

```ts
// BAD — plain values, not reactive
const { count, name } = reactive({ count: 0, name: 'test' })

// GOOD
const state = reactive({ count: 0, name: 'test' })
// or use ref() instead
const count = ref(0)
```

### Refs in arrays/Maps/Sets require .value

Refs auto-unwrap inside `reactive()` objects, but NOT in arrays, Maps, or Sets:

```ts
const state = reactive({ count: ref(0) })
state.count++ // no .value needed

const arr = reactive([ref(1)])
arr[0].value // .value IS needed

const map = reactive(new Map([['key', ref(1)]]))
map.get('key').value // .value IS needed
```

### markRaw for non-reactive instances

Library instances (maps, editors, chart instances) break when made reactive:

```ts
// BAD — proxy wrapping breaks internal methods
const map = ref(new mapboxgl.Map(options))

// GOOD
import { markRaw } from 'vue'
const map = shallowRef(markRaw(new mapboxgl.Map(options)))
```

### Proxy identity hazard

```ts
const raw = {}
const state = reactive({ nested: raw })

state.nested === raw // false — state.nested is a Proxy
state.nested === state.nested // true
```

Use `toRaw()` when you need to compare with the original object.

## Computed

### No side effects in computed

Computed getters must be pure. No API calls, no mutations, no DOM manipulation:

```ts
// BAD
const data = computed(() => {
  fetch('/api/data') // side effect!
  return transform(raw.value)
})

// GOOD — use watch or watchEffect for side effects
const data = computed(() => transform(raw.value))
watch(raw, () => fetch('/api/data'))
```

### Computed returning objects triggers effects every time

Computed properties returning new objects/arrays create new references on each access, triggering unnecessary watchers:

```ts
// BAD — new array reference every time
const filtered = computed(() => items.value.filter(i => i.active))

// Acceptable for template rendering (Vue optimizes this)
// But watch(filtered, ...) fires on every dependency change
// even if the actual items haven't changed
```

### Conditional dependencies

Dependencies inside conditional branches are only tracked when the condition is true:

```ts
// BAD — dep not tracked when condition is false
watchEffect(() => {
  if (condition.value) {
    console.log(dep.value) // only tracked when condition=true
  }
})

// GOOD — explicit watch for conditional deps
watch([condition, dep], ([cond, d]) => {
  if (cond) console.log(d)
})
```

## Watchers

### Deep watch performance

`{ deep: true }` on large objects is expensive. Use depth limit (Vue 3.5+) or watch specific paths:

```ts
// BAD — watches everything
watch(largeState, callback, { deep: true })

// GOOD — limit depth
watch(largeState, callback, { deep: 2 })

// GOOD — watch specific getter
watch(() => largeState.value.specificField, callback)
```

### Dependencies after await are not tracked

`watchEffect` stops tracking after the first `await`:

```ts
// BAD — changes to dep2 are NOT tracked
watchEffect(async () => {
  console.log(dep1.value) // tracked
  await someAsyncWork()
  console.log(dep2.value) // NOT tracked
})

// GOOD — read all deps before await
watchEffect(async () => {
  const d1 = dep1.value // tracked
  const d2 = dep2.value // tracked
  await someAsyncWork()
  // use d1, d2
})
```

### Flush timing for DOM access

Default watchers run before DOM updates. Use `flush: 'post'` to access updated DOM:

```ts
watch(source, () => {
  // DOM is updated here
  nextTick() // not needed with flush: 'post'
}, { flush: 'post' })
```

## Props

### Passing props to composables loses reactivity

```ts
// BAD — static value, composable won't react to changes
const result = useMyComposable(props.id)

// GOOD — pass as getter
const result = useMyComposable(() => props.id)

// GOOD — pass as toRef
const result = useMyComposable(toRef(props, 'id'))
```

### Boolean prop casting

Vue casts boolean props differently than you might expect:

```ts
defineProps<{ disabled?: boolean }>()
```

```vue
<MyComponent disabled />     <!-- true -->
<MyComponent :disabled="true" /> <!-- true -->
<MyComponent />              <!-- undefined, NOT false -->
```

If you need `false` as default, declare it explicitly in `withDefaults` or destructuring defaults.

## TypeScript

### Template type casting

Vue templates don't support TypeScript `as` syntax. Use computed or type guards:

```ts
// BAD — won't work in template
// {{ (item as User).name }}

// GOOD — computed
const typedItem = computed(() => item.value as User)
// {{ typedItem.name }}
```

### shallowRef for dynamic component storage

Storing component references in `ref()` wraps them in a deep reactive proxy, causing performance issues:

```ts
// BAD
const currentComponent = ref(MyComponent)

// GOOD
const currentComponent = shallowRef(MyComponent)
```

## Tailwind

### Dynamic class names are invisible to Tailwind

Tailwind uses static analysis at build time. Dynamically constructed class names are not detected:

```vue
<!-- BAD — classes missing in production -->
<div :class="`bg-${color}-500`">

<!-- GOOD — mapping object with complete class names -->
<script setup lang="ts">
const colorClasses: Record<string, string> = {
  red: 'bg-red-500',
  blue: 'bg-blue-500',
  green: 'bg-green-500'
}
</script>

<div :class="colorClasses[color]">
```

## Performance

### Virtualize large lists

Rendering 100+ items causes DOM performance issues. Use virtual scrolling:

```vue
<!-- Use libraries like @tanstack/vue-virtual or vue-virtual-scroller -->
```

### v-once and v-memo for static content

```vue
<!-- Render once, skip future updates -->
<span v-once>{{ expensiveComputation() }}</span>

<!-- Memoize — only re-render when id changes -->
<div v-memo="[item.id]">
  {{ expensiveFormat(item) }}
</div>
```

### Props stability

Avoid passing new object/array references as props on every render:

```vue
<!-- BAD — new array every render -->
<Child :items="rawItems.filter(i => i.active)" />

<!-- GOOD — stable computed reference -->
<Child :items="activeItems" />
```

<!--
Source references:
- https://vuejs.org/guide/best-practices/performance.html
- https://vuejs.org/guide/extras/reactivity-in-depth.html
- https://github.com/vuejs-ai/skills
-->
