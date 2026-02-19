---
name: patterns
description: Component communication patterns — async emit callbacks, v-model patterns, slot usage, and loading state management
---

# Component Patterns

## Async Emit with Callback

For operations requiring async work in the parent (save, delete, duplicate), use the callback pattern. The child manages its own loading state.

### Child Component

```ts
const emit = defineEmits<{
  save: [data: FormData, done: (error?: string) => void]
  delete: [done: (error?: string) => void]
}>()

const isSaving = ref(false)

function handleSave(): void {
  isSaving.value = true
  emit('save', formData, (error?: string) => {
    isSaving.value = false
    if (error) {
      toast.error('Save failed', { description: error })
    }
  })
}
```

### Parent Component

```ts
async function handleSave(
  data: FormData,
  done: (error?: string) => void,
): Promise<void> {
  const result = await saveToBackend(data)
  if (result.isErr()) {
    done(result.error.message)
    return
  }
  done()
}
```

### Template

```vue
<!-- Child manages loading internally -->
<Button :disabled="isSaving" @click="handleSave">
  <Loader2Icon v-if="isSaving" class="h-4 w-4 animate-spin" />
  {{ isSaving ? 'Saving...' : 'Save' }}
</Button>

<!-- Parent just passes the handler -->
<ChildComponent @save="handleSave" />
```

This pattern keeps the child self-contained (owns its loading state), the parent focused on business logic, and avoids passing loading state props back and forth.

## v-model Patterns

### Single v-model

```vue
<!-- Parent -->
<SearchInput v-model="query" />

<!-- Child -->
<script setup lang="ts">
const model = defineModel<string>({ required: true })
</script>

<template>
  <input v-model="model" />
</template>
```

### Multiple v-models

```vue
<!-- Parent -->
<UserForm v-model:first-name="user.firstName" v-model:age="user.age" />

<!-- Child -->
<script setup lang="ts">
const firstName = defineModel<string>('firstName')
const age = defineModel<number>('age')
</script>
```

### v-model with Modifiers

```vue
<!-- Parent -->
<TextInput v-model.trim.capitalize="text" />

<!-- Child -->
<script setup lang="ts">
const [model, modifiers] = defineModel<string>()

function handleInput(value: string) {
  let result = value
  if (modifiers.trim) result = result.trim()
  if (modifiers.capitalize) result = result.charAt(0).toUpperCase() + result.slice(1)
  model.value = result
}
</script>
```

## Slot Patterns

Always use shorthand `#name` syntax:

```vue
<Card>
  <template #header>
    <h2>Title</h2>
  </template>
  <template #default>
    Content
  </template>
</Card>
```

### Scoped Slots

```vue
<!-- Parent provides renderer -->
<DataTable :items="users">
  <template #row="{ item, index }">
    <span>{{ index }}: {{ item.name }}</span>
  </template>
</DataTable>

<!-- Child exposes data to slot -->
<script setup lang="ts">
defineSlots<{
  row(props: { item: User; index: number }): any
}>()
</script>

<template>
  <div v-for="(item, index) in items" :key="item.id">
    <slot name="row" :item :index />
  </div>
</template>
```

### Conditional Slot Rendering

Only render wrapper when slot content is provided:

```vue
<template>
  <div>
    <header v-if="$slots.header">
      <slot name="header" />
    </header>
    <slot />
  </div>
</template>
```

## Loading State Patterns

### Simple boolean

```ts
const isLoading = ref(false)

async function fetchData() {
  isLoading.value = true
  try {
    // ...
  } finally {
    isLoading.value = false
  }
}
```

### Multiple operations with record

```ts
const loading = ref<Record<string, boolean>>({})

async function deleteItem(id: string) {
  loading.value[id] = true
  try {
    await api.delete(id)
  } finally {
    delete loading.value[id]
  }
}

// Template: :disabled="loading[item.id]"
```
