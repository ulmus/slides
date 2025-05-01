---
title: Vue 3 Reactivity Deep Dive
layout: cover
---

# Vue 3 Reactivity Deep Dive

Understanding how Vue tracks and updates state

---

## Agenda

1. Reactivity Fundamentals
2. Reactive vs. Non-Reactive Values
3. Shallow Reactivity
4. Compiler Macros in `<script setup>`
5. Common Reactivity Pitfalls
6. Advanced Topics Overview
7. Q&A / Discussion

---

## Reactivity Fundamentals

- Vue tracks dependencies during rendering
- Changes to reactive data trigger updates
- `computed`, `watch`, and templates rely on this system

```ts {monaco-run}
import {computed, watch, ref } from 'vue'
const count = ref(0);

const double = computed<number>(() => count.value * 2);

watch(count, (newVal) => {
  console.log("count changed to", newVal);
});

watch(double, (newVal) => {
  console.log("double count changed to", newVal);
});
```


<!-- 
The reactivuty is dependent on triggering events. In the flow of normal template rendering, you don't have to think too much of that, but in practice. Vue collects all changes in a `tick` and then
-->

---

## How Vue Tracks Changes

- Vue uses proxies to intercept access and mutation
- Dependencies are tracked during reactive reads
- Updates trigger a reactive "effect" to re-run

```ts {monaco-run}
import { watchEffect, reactive } from 'vue'

const state = reactive({ msg: "hello" });

watchEffect(() => {
  console.log(state.msg); // tracked as a dependency
});
```

---

## `computed` vs `watch`

- `computed`: derived state, memoized
- `watch`: side effects, async, or reacting to multiple sources

```ts
const price = ref(100);
const tax = ref(0.25);

const total = computed(() => price.value * (1 + tax.value)); // recalculates only when needed

watch([price, tax], ([newP, newT]) => {
  console.log("Price or tax changed", newP, newT);
});
```

---

## Reactive vs. Non-Reactive Values

- Primitives: use `ref()`
- Objects/arrays: use `reactive()`

```ts
const count = ref(0); // primitive
const user = reactive({ name: "Ada" }); // object

user.name = "Alan"; // triggers updates
```

- `Map`, `Set`, `WeakMap`, `WeakSet`: **partially reactive**

---

## Why Some Values Aren't Reactive

- JavaScript Proxies can't trap some operations
- Arrays: reactivity works for indexed access and mutation
- Adding properties to reactive objects: use `Vue.set()` (in Vue 2) or ensure property exists (Vue 3)

---

## Advanced Reactivity: Shallow APIs

- Root-level reactivity only
- Useful for performance or when deep tracking causes issues

```ts
const state = shallowReactive({
  nested: { count: 0 },
});

state.nested.count++; // this change is NOT reactive
```

---

## Compiler Macros in `<script setup>`

```vue
<script setup>
const props = defineProps<{ title: string }>()
const emit = defineEmits<['close']>()

defineExpose({ doSomething })
</script>
```

- Only available in `<script setup>`
- Compile-time sugar only — no runtime import

---

## What Compiler Macros Actually Do

- Replace boilerplate from Options API and classic Setup API
- Enable stronger type inference

```ts
// Instead of:
defineProps();
// Compiler transforms to:
const __props = defineProps(); // injected by SFC compiler
```

---

## Common Reactivity Pitfalls

### ❌ Destructuring reactive objects:

```ts
const state = reactive({ count: 1 });
const { count } = state; // loses reactivity!
```

### ✅ Correct approach:

```ts
const { count } = toRefs(state);
```

---

## More Pitfalls

- Nested objects in `reactive()` are reactive — but only if accessed properly
- Using `ref().value` incorrectly in templates

```ts
const user = reactive({ name: "Ada" });
// reactivity lost:
const name = user.name;

// solution:
const { name } = toRefs(user);
```

---

## Avoiding Reactivity Pitfalls

- Always use `toRefs()` when destructuring
- Avoid mutating reactive objects outside their reactive scope
- Prefer `ref()` for single values, `reactive()` for structures

---

## Advanced Reactivity Topics (Optional Deep Dive)

- `effectScope()` – group effects for scoped teardown
- `customRef()` – create refs with custom behavior (e.g., debounce)
- `readonly()` / `shallowReadonly()` – safe public APIs
- `markRaw()` – exclude objects from reactivity
- Reactivity Transform – avoid `.value` (experimental)
- Understanding reactivity graph, lazy updates
- Memory management for watchers/effects

---

## Summary

- Vue's reactivity is proxy-based and declarative
- `ref`, `reactive`, `computed`, and `watch` serve distinct roles
- Use advanced tools and patterns when complexity increases

---

## Questions & Discussion

What issues or unexpected behavior have you encountered with Vue reactivity?
