---
title: Vue 3 Reactivity Deep Dive
transition: fade
layout: cover
theme: light-icons
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
import { computed, watch, ref } from "vue";
const count = ref(0);

const double = computed<number>(() => count.value * 2);

watch(count, (newVal) => {
  console.log("count changed to", newVal);
});

watch(double, (newVal) => {
  console.log("double count changed to", newVal);
});

count.value = 1;
```

<!--
The reactivity is dependent on triggering events. In the flow of normal template rendering, you don't have to think too much of that, but in practice. Vue collects all changes in a `tick` and then executes them all at once, for efficiency.
Let's explore this and how the watchers interact with this.
-->

---

## How Vue Tracks Changes

- Vue uses proxies to intercept access and mutation
- Dependencies are tracked during reactive reads (inside `watchEffect`, `computed`, etc)
- Updates trigger a reactive "effect" to re-run
- Updates are batched in "ticks"

```ts {monaco-run}
import { watchEffect, reactive } from "vue";

const state = reactive({ msg: "hello" });

watchEffect(() => {
  console.log(state.msg); // tracked as a dependency
});
```

---

## `computed` vs `watch`

- `computed`: derived state, memoized
- `watch`: side effects, async, or reacting to multiple sources

```ts {monaco-run}
import { computed, watch, ref } from "vue";

const price = ref(100);
const tax = ref(0.25);

const total = computed(() => price.value * (1 + tax.value)); // recalculates only when needed

watch([price, tax], ([newP, newT]) => {
  console.log("Price or tax changed", newP, newT);
},{
  immediate: true
});
```

---

## Reactive vs. Non-Reactive Values

- Primitives: use `ref()`
- Objects/arrays: use `reactive()`
- This is to enable interception of reads

```ts {monaco-run}
import { ref, reactive } from "vue";

const count = ref(0); // primitive
const user = reactive({ name: "Ada" }); // object

user.name = "Alan"; // triggers updates
```

- `Map`, `Set`, `WeakMap`, `WeakSet`: **partially reactive**

---

## Why Some Values Aren't Reactive

- JavaScript Proxies can't trap some operations
- Arrays: reactivity works for indexed access and mutation
- You can't add reactive properties to reactive objects, ensure property exists when calling `reactive()`

---

## General guideline: use `ref()` unless you absolutely need `reactive()`
- `ref()` is simpler and more predictable
- `reactive()` is for complex objects or when you need deep reactivity
-  If you can treat objects and arrays as immutable, prefer `ref()`, but remember, then you can't update the properties of the object or content of the array.
- You can use the `Readonly` TypeScript type to ensure that the object is not mutated.

```ts {monaco}
import { ref } from "vue";
const user = ref<Readonly<Record<string, string>>>({ name: "Ada" }); // use ref for immutability
user.value.name = "Alan"; // this is not allowed
```
---

## Advanced Reactivity: Shallow APIs

- Root-level reactivity only
- Useful for performance or when deep tracking causes issues
- Specifically good for tracking immutable data structures

```ts {monaco-run}
import { shallowRef } from "vue";

const state = shallowRef<Readonly<{ nested: { count: number } }>>({
  nested: { count: 0 },
});

state.nested.count++; // this change is NOT reactive
```

---

## Compiler Macros in `<script setup>`
- `defineProps()`, `defineEmits()`, `defineExpose()` (and more)
- Provide compile-time helpers for props, emits, and exposing methods
- Only available in `<script setup>`
- Compile-time sugar only — no runtime import
- The props can be destructured, but the destructuring will be replaced with the original name of the prop.

```vue {monaco}
<script setup>
const { title } = defineProps<{ title: string }>()
const allCaps = computed(() => title.toUpperCase())
/*
 This will be compiled to:
const __props = defineProps<{ title: string }>()
const allCaps = computed(() => __props.title.toUpperCase())
*/
</script>
```
---

## Common Reactivity Pitfalls
- Remember, reactivity is tracked at the time of access
- The access needs to be inside a reactive context (like `watchEffect`, `computed`, etc.)
- Passing an object member to a function will lose reactivity
- Destructuring reactive objects (like `props`) directly will lose reactivity, use `toRefs()`

### Destructuring reactive objects:

```ts
const state = reactive({ count: 1 });
const { count } = state; // loses reactivity!
const { count: reactiveCount } = toRefs(state); // use toRefs for reactivity
```

### Passing reactive properties to functions:

```ts
const state = reactive({ count: 1 });
const logCount = (count) => console.log(count);
logCount(state.count); // count is accessed outside a reactive context and is not reactive here
watchEffect(() => {
  logCount(state.count); // here count is accessed in a reactive context and is reactive
});
```
---

## Avoiding Reactivity Pitfalls

- Prefer `ref()` and setting `.value` for simplicity unless you need deep reactivity
- Always use `toRefs()` when destructuring reactive objects
- Be aware of the context in which you access reactive properties


---

## Advanced Reactivity Topics (Optional Deep Dive)

- `effectScope()` – group effects for scoped teardown
- `customRef()` – create refs with custom behavior (e.g., debounce)
- `computed()` with `setter` – computed properties with side effects
- `readonly()` / `shallowReadonly()` – safe public APIs
- `markRaw()` – exclude objects from reactivity (good for dynamic component references)
- `nextTick()` – wait for DOM updates after reactivity changes
- Understanding reactivity graph, lazy updates
- Memory management for watchers/effects

---
layout: center
---

## Summary

- Vue's reactivity is proxy-based and declarative
- Reactivity is tracked in a reactive context at access time
- Use `ref()` unless you need deep reactivity

---
layout: end
---

## Questions & Discussion

What issues or unexpected behavior have you encountered with Vue reactivity?
