---
name: macros
description: Vue 3 script setup compiler macros — defineProps, defineEmits, defineModel, defineExpose, defineOptions, defineSlots, and generic components
---

# Script Setup Macros

## defineProps

Type-based declaration with explicit interface:

```ts
interface Props {
  title: string
  count?: number
  items: string[]
}

const props = defineProps<Props>()
```

With defaults (Vue 3.5+, destructured):

```ts
const { title, count = 0 } = defineProps<{
  title: string
  count?: number
}>()
```

With defaults (Vue 3.4 and below):

```ts
const props = withDefaults(defineProps<{
  title: string
  items?: string[]
}>(), {
  items: () => []
})
```

## defineEmits

Named tuple syntax (Vue 3.3+):

```ts
const emit = defineEmits<{
  update: [value: string]
  change: [id: number, name: string]
  close: []
}>()

emit('update', 'new value')
emit('change', 1, 'name')
emit('close')
```

## defineModel

Two-way binding via `v-model`. Available in Vue 3.4+.

```ts
// Basic — creates "modelValue" prop
const model = defineModel<string>()
model.value = 'hello' // emits "update:modelValue"

// Named model — consumed via v-model:count
const count = defineModel<number>('count', { default: 0 })

// With modifiers
const [value, modifiers] = defineModel<string>()
if (modifiers.trim) {
  // handle trim modifier
}

// With transformers
const [value, modifiers] = defineModel({
  get(val) { return val?.toLowerCase() },
  set(val) { return modifiers.trim ? val?.trim() : val }
})
```

Parent usage:

```vue
<Child v-model="name" />
<Child v-model:count="total" />
<Child v-model.trim="text" />
```

Use `required: true` to prevent double-emit during initialization:

```ts
const model = defineModel<Item>({ required: true })
```

## defineExpose

Components are closed by default. Explicitly expose properties for template ref access:

```ts
const count = ref(0)
const reset = () => { count.value = 0 }

defineExpose({ count, reset })
```

Parent access:

```ts
const childRef = ref<{ count: number; reset: () => void }>()
childRef.value?.reset()
```

## defineOptions

Declare component options without a separate `<script>` block. Vue 3.3+.

```ts
defineOptions({
  inheritAttrs: false,
  name: 'CustomName'
})
```

## defineSlots

Typed slot props. Vue 3.3+.

```ts
const slots = defineSlots<{
  default(props: { item: string; index: number }): any
  header(props: { title: string }): any
}>()
```

## Generic Components

Use the `generic` attribute for type-safe reusable components:

```vue
<script setup lang="ts" generic="T extends string | number">
defineProps<{
  items: T[]
  selected: T
}>()
</script>
```

Multiple generics with constraints:

```vue
<script setup lang="ts" generic="T, U extends Record<string, T>">
defineProps<{
  data: U
  key: keyof U
}>()
</script>
```

## Local Custom Directives

Use `vNameOfDirective` naming convention:

```ts
const vFocus = {
  mounted: (el: HTMLElement) => el.focus()
}
```

```vue
<template>
  <input v-focus />
</template>
```

<!--
Source references:
- https://vuejs.org/api/sfc-script-setup.html
-->
