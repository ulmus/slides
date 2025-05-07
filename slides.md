---
title: Vue 3 Reactivity Deep Dive
transition: fade
layout: image-right
theme: default
author: Jens Alm
date: 2025-05-07
description: Understanding how Vue tracks and updates state
tags: [vue, reactivity, vue3]
image: /industrial-process-cogs-and-wheels.png
---

# Vue 3 Reactivity Deep Dive

Understanding how Vue tracks and updates state

---

## Agenda

<Toc mode="all" columns="2" />

---

# Reactivity Fundamentals

- Vue uses proxies to intercept access and mutation
- Changes to reactive data trigger updates in batches ("ticks")
- `computed`, `watch`, and templates rely on this system
- Dependencies are tracked during reactive reads (inside `watchEffect`, `computed`, etc)
- Updates trigger a reactive "effect" to re-run

---

## Basic Example
```ts {monaco-run}
import { computed, watch, ref } from "vue";

// Create a reactive reference
const count = ref(0);

// Create a computed property that depends on count
const double = computed<number>(() => count.value * 2);

// Watch for changes in count and double
watch(count, (newVal) => {
  console.log(`count changed to ${newVal}`);
});

watch(double, (newVal) => {
  console.log(`double count changed to ${newVal}`);
});
```

<!--
The reactivity is dependent on triggering events. In the flow of normal template rendering, you don't have to think too much of that, but in practice. Vue collects all changes in a `tick` and then executes them all at once, for efficiency.
Let's explore this and how the watchers interact with this.
-->

---

## `computed` vs `watch` vs `watchEffect`

- `computed`: derived state, memoized, recalculates only when dependencies change
- `watch`: trigger side effects on changes, can watch multiple sources, runs after DOM updates
- `watchEffect`: auto-tracks dependencies, runs immediately, behaves like `computed` but for side effects

```ts {monaco-run}
import { computed, watch, watchEffect, ref } from "vue";

const price = ref(100);
const tax = ref(0.25);

const total = computed(() => price.value * (1 + tax.value)); // recalculates only when needed

watch([price, tax], ([newP, newT]) => {
  console.log("Price or tax changed", newP, newT);
});

watchEffect(() => {
  console.log(`Total price is ${total.value}`); // runs immediately and tracks dependencies
});
```

---

## Making Values Reactive

- Primitives: use `ref()`
- Objects/arrays/maps/sets: use `reactive()`
- This is to enable interception of reads

```ts {monaco-run}
import { ref, reactive } from "vue";

const count = ref(0); // primitive
const user = reactive({ name: "Ada" }); // object

user.name = "Alan"; // triggers updates
```

---

## Special Cases

- `WeakMap`, `WeakSet`: partially reactive, due to limitations of JavaScript Proxies
- `readonly()`: creates a read-only version of a reactive object
- `shallowReactive()` and `shallowReadonly()` : create a shallow version of reactive/readonly (only root-level properties are reactive)
- `toRefs()`: convert a reactive object to refs, useful for destructuring
- `toRaw()`: get the original object from a reactive proxy
- `effectScope()`: create a scope for effects, useful for cleanup and grouping

---

## Why Some Values Aren't Reactive

- JavaScript proxies can only track objects, not primitives
- You can't add reactive properties to reactive objects, ensure property exists when calling `reactive()`
- `WeakMap` and `WeakSet`: not fully reactive due to garbage collection behavior


---

## Objects In Refs

- `ref()` creates a reactive reference to a primitive or object
- When you wrap an object with `ref()`, Vue will make it deeply reactive, using `reactive()` under the hood.

```ts {monaco-run}
import { ref, watchEffect } from "vue";
const state = ref({
  nested: { count: 0 },
});
watchEffect(() => {
  console.log(state.value.nested.count);
});
state.value.nested.count++; // this change is reactive
```



---

## Shallow APIs

- `shallowReactive()` and `shallowRef()`
- Root-level reactivity only
- Useful for performance or when deep tracking causes issues
- Specifically good for tracking immutable data structures

```ts {monaco-run}
import { shallowRef, watchEffect } from "vue";

const state = shallowRef({
  nested: { count: 0 },
});

watchEffect(() => {
  console.log(state.value.nested.count);
});

state.value.nested.count++; // this change is NOT reactive
state.value = { nested: { count: 1 } }; // this change IS reactive
```

---

## General guideline: use `ref()` or `shallowRef()` unless you absolutely need `reactive()`
- `ref()` is simpler and more predictable
- When you wrap an object with `ref()`, Vue will make it deeply reactive, using `reactive()` under the hood.
- `shallowRef()` is a shallow version of `ref()`, only the root-level properties are reactive. This is usefule to reduce complexity and performance overhead when you don't need deep reactivity.
- `reactive()` is for complex objects or when you need deep reactivity, replacing a reactive object with a new one will not trigger reactivity.

---

# Corner Cases And Useful Patterns

<Toc mode="onlyCurrentTree" />

---

## Common Reactivity Pitfalls
- Remember, reactivity is tracked at the time of access
- The access needs to be inside a reactive context (like `watchEffect`, `computed`, etc.)
- Take note of where a variable is actually accessed.
- Destructuring reactive objects (like `props`) directly will lose reactivity, use `toRefs()` or `defineProps()` instead
- Reassigning a variable that is a reactive reference will not trigger reactivity, use `.value` to update the value
- Using `v-model` with a non-reactive reference will not trigger reactivity, use `ref()` or `reactive()` to make it reactive

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

### Destructuring reactive objects:

```ts
const state = reactive({ count: 1 });
const { count } = state; // loses reactivity!
const { count: reactiveCount } = toRefs(state); // use toRefs for reactivity
```

---

### Passing reactive properties to functions:

```ts {monaco-run}
import { reactive, watch, watchEffect, ref } from "vue";

const state = reactive({ count: 1 });
const logCount = (text: string, count: number) => console.log(`${text} ${count}`);

logCount("Accessed directly in setup:", state.count);

watchEffect(() => {
  logCount("Accessed in watchEffect:", state.count);
});

watch(() => state.count, (newCount) => logCount("Accessed in direct watch:", newCount), {immediate: true});

const countRef = ref(state.count);
watch(countRef, (newCount) => logCount("Accessed in watch via new ref:", newCount), {immediate: true});
```
---

## Avoiding Reactivity Pitfalls

- Prefer `ref()` or `shallowRef()` and setting `.value` for simplicity unless you need deep reactivity
- Always use `toRefs()` when destructuring reactive objects
- Be aware of the context in which you access reactive properties


---

## Advanced Reactivity Topics (Optional Deep Dive)

- `customRef()` – create refs with custom behavior (e.g., debounce)
- `computed()` with `setter` – computed properties with side effects
- `nextTick()` – wait for DOM updates after reactivity changes
- `markRaw()`: mark an object as non-reactive, useful for performance optimizations
- `effectScope()`: create a scope for effects, useful for cleanup and grouping
- `onBeforeUnmount()` and `onUnmounted()` – lifecycle hooks for cleanup
- use composition API to create reusable reactive logic

---
layout: center
---

## Summary

- Vue's reactivity is proxy-based and declarative
- Reactivity is tracked in a reactive context at access time
- Use `ref()` or `shallowRef()` unless you need deep reactivity

---
layout: end
---

## Questions & Discussion

What issues or unexpected behavior have you encountered with Vue reactivity?
