# RFC: Roxbury ↔ Common Controls UI Alignment Roadmap

**Author:** Saul Graves (Roxbury Design System)  
**Date:** October 2025  
**Status:** Draft for Review  
**Audience:** Roxbury Design System, Cirrus (Common Controls), SPA Development Teams

---

## 1. Background

Two separate Vue-based UI libraries currently exist within the organization:

- **Roxbury (`@cnodigital/roxbury`)** — centralized design system providing Vue components, Sass-based styles, and icons.
- **Common Controls (`cno-common-controls`)** — a Cirrus-managed library that extends Roxbury’s design language but adds application-specific logic and services (Pinia stores, i18n, entitlements, API calls, etc.).

This duplication has led to:
- Diverging UI implementations and inconsistent UX.
- Version drift and mismatched styles (Roxbury v5 vs Common Controls using Roxbury v3 styles).
- Conflicts and visual regressions during dependency updates.

---

## 2. Goal

Unify the **UI layer** under Roxbury while allowing Common Controls to remain a **services and integration layer**.  
All presentation and styling should originate in Roxbury.

---

## 3. Non-Goals

- Roxbury will **not** include business logic, API calls, or translation layers.
- Common Controls will **not** be deprecated—it will remain the integration library.
- No breaking changes for SPAs currently using Common Controls.

---

## 4. Adoption Criteria (Quality Gates)

| Category | Success Criteria |
|-----------|------------------|
| **Performance** | No measurable degradation in per-component bundle size (<2% increase allowed). |
| **Compatibility** | No breaking changes to Common Controls’ public APIs. |
| **Testing** | All existing unit and Playwright E2E tests pass. |
| **Accessibility** | Meets Roxbury’s existing accessibility standards (`axe-playwright` compliance). |
| **Visual Regression** | No unintended visual differences across pilot SPAs. |
| **Web Vitals** | No degradation in FCP or LCP in production telemetry. |

---

## 5. Technical Approach

### 5.1 Import Strategy

Common Controls components should import Roxbury components directly from **source modules**, not the bundled entry:

```js
import { RoxInputCore } from '@cnodigital/roxbury/src/components'
```

This ensures tree-shaking and minimal payload.

### 5.2 Styles Integration

Common Controls continues consuming **per-component CSS** (e.g. `Input.min.css`)  
and avoids importing `Core.min.css` globally.

### 5.3 Layout Refactor Strategy

- Move UI-only Layout, Header, Footer components into Roxbury.
- Retain logic in Common Controls (user injection, entitlement checks).
- Use Roxbury components for layout visuals; pass computed props from CC.

### 5.4 Incremental Replacement Plan

1. Audit Common Controls components overlapping Roxbury.
2. Replace low-risk ones first (`Button`, `Card`, `Modal`, `Layout`).
3. Validate bundle size and test results after each migration.
4. Gate merges on passing all validations.

---

## 6. Validation Pipeline

| Step | Tool | Owner |
|------|------|--------|
| Bundle size diff | `vite-bundle-analyzer` or `rollup-plugin-visualizer` | Roxbury |
| Unit test suite | Vitest | Common Controls |
| E2E regression | Playwright | Common Controls |
| Accessibility | `axe-playwright` | Roxbury |
| Visual diffs | Playwright screenshots | Roxbury |
| Web vitals monitoring | SPA analytics dashboards | Cirrus |

---

## 7. Release Coordination

- Roxbury publishes alpha tags (`v6.0.0-alpha.x`).
- Common Controls consumes alpha builds in CI for pre-validation.
- Shared Azure DevOps org: run Roxbury build → CC integration tests → analyzer → publish.

---

## 8. Risks & Mitigations

| Risk | Mitigation |
|------|-------------|
| Bundle bloat | Tree-shakeable source imports; Core + Plugins design. |
| Breaking props | Adapter shims in Common Controls. |
| Visual regressions | Snapshot tests. |
| Dependency drift | Pin to Roxbury minor versions (`~6.x`). |

---

## 9. Milestones

| Phase | Timeline | Focus |
|--------|-----------|--------|
| **1 – Alignment RFC Approval** | Weeks 1–2 | Approve plan, create audit table. |
| **2 – Infrastructure Setup** | Weeks 2–3 | Add analyzer step to pipelines. |
| **3 – Layout Integration Pilot** | Weeks 3–6 | Replace Layout/Header/Footer. |
| **4 – Component Parity Audit** | Weeks 6–8 | Replace simple UI components. |
| **5 – Broader Rollout** | Weeks 8–12 | Migrate all shared UI elements. |
| **6 – Continuous Sync** | Ongoing | Add version lock, regression tests. |

---

## 10. Next Steps

1. Generate component audit matrix (below).  
2. Approve target replacements.  
3. Publish Roxbury alpha.  
4. Integrate analyzer and test steps in CI.  
5. Conduct pilot migration (Layout/Header/Footer).

---

# Component Audit Table Template

| Component (Common Controls) | SPA Usage | CC JS Size (KB) | CC CSS (KB) | Roxbury Component | Rox Import Path | Rox JS (KB) | Rox CSS (KB) | Δ JS | Δ CSS | API Match | Adapter Complexity | Visual Risk | Perf Risk | A11y Risk | Tests | Pilot SPA | Owner | Status | Notes |
|-----------------------------|------------|-----------------|--------------|-------------------|-----------------|--------------|--------------|-------|-------|------------|--------------------|--------------|------------|-----------|--------|------------|--------|--------|--------|
| Layout | 6 | 3.2 | 1.1 | Layout, Header, Footer | @cnodigital/roxbury/src/components | 4.0 | 1.3 | +0.8 | +0.2 | Adapter | Med | Med | Low | Low | Unit/E2E | App A,B | CC | Planned | Replace markup with Rox Layout; retain logic. |
| Header | 6 | 1.0 | 0.5 | Header | @cnodigital/roxbury/src/components | 1.2 | 0.6 | +0.2 | +0.1 | Yes | Low | Low | Low | Low | Unit | App A | CC | Planned | i18n text provided by CC. |
| Footer | 6 | 0.9 | 0.4 | Footer | @cnodigital/roxbury/src/components | 1.0 | 0.5 | +0.1 | +0.1 | Yes | Low | Low | Low | Low | Unit | App B | CC | Planned | Links + legal text from CC. |
| Input | 8 | 1.6 | 0.3 | RoxInputCore (+plugins) | @cnodigital/roxbury/src/components | 2.8 | 0.35 | +1.2 | +0.05 | Adapter | Med | Med | Med | Low | Unit/E2E/A11y | App C | Rox | Planned | Migrate to Core + Plugins structure. |

---

# Guide: Refactoring Components to Core + Plugins Architecture

## 1. Concept

Split large, feature-rich components into:

- **Core Component:**  
  Minimal, purely visual + v-model behavior, zero optional features.

- **Plugins (Composables):**  
  Self-contained logic (formatting, masking, validation, search, etc.), imported only when needed.

- **Full Back-Compat Component:**  
  Combines Core + default plugins to preserve existing APIs for current Roxbury consumers.

This allows:
- Small bundle sizes via tree-shaking.  
- Centralized UI styles and tokens.  
- Common Controls to import *only* what’s needed.

---

## 2. RoxInput Example

### 2.1 `RoxInputCore.vue` (minimal)

```vue
<template>
  <div class="rox-input">
    <input
      :id="id"
      :name="name"
      :type="type"
      :value="modelValue"
      :disabled="disabled"
      :readonly="readonly"
      :aria-invalid="invalid || undefined"
      :aria-describedby="ariaDescribedby"
      @input="e => $emit('update:modelValue', e.target.value)"
      @blur="$emit('blur')"
      @focus="$emit('focus')"
      class="rox-input__control"
    />
    <slot name="suffix" />
  </div>
</template>

<script setup>
defineProps({
  modelValue: [String, Number],
  id: { type: String, default: undefined },
  name: { type: String, default: undefined },
  type: { type: String, default: 'text' },
  disabled: { type: Boolean, default: false },
  readonly: { type: Boolean, default: false },
  invalid: { type: Boolean, default: false },
  ariaDescribedby: { type: String, default: undefined },
})
defineEmits(['update:modelValue', 'blur', 'focus'])
</script>
```

---

### 2.2 Plugins (composables)

#### `usePasswordMask.js`
```js
import { ref, computed } from 'vue'

export function usePasswordMask(initial = true) {
  const masked = ref(initial)
  const inputType = computed(() => (masked.value ? 'password' : 'text'))
  const toggle = () => { masked.value = !masked.value }
  const label = computed(() => (masked.value ? 'Show password' : 'Hide password'))
  return { masked, inputType, toggle, label }
}
```

#### `useFormatter.js`
```js
import { ref, watch } from 'vue'

export function useFormatter({ modelValue, emit, format, parse }) {
  const display = ref(format(modelValue))

  watch(() => modelValue.value ?? modelValue, (val) => {
    display.value = format(val)
  })

  function onInput(nextDisplay) {
    const raw = parse(nextDisplay)
    emit('update:modelValue', raw)
  }

  return { display, onInput }
}
```

#### `useValidation.js`
```js
import { computed } from 'vue'

export function useValidation({ modelValue, rules = [] }) {
  const errors = computed(() => {
    for (const rule of rules) {
      const msg = rule(modelValue.value ?? modelValue)
      if (msg) return [msg]
    }
    return []
  })
  const invalid = computed(() => errors.value.length > 0)
  return { errors, invalid }
}
```

*(You can add other tiny composables like `useSearch()` or `useStatus()` with the same pattern.)*

---

### 2.3 Full `RoxInput.vue` (back‑compat composed component)

```vue
<template>
  <RoxInputCore
    :id="id"
    :name="name"
    :type="resolvedType"
    :model-value="display"
    :disabled="disabled"
    :readonly="readonly"
    :invalid="invalid"
    :aria-describedby="ariaDescribedby"
    @update:model-value="onInput"
    @blur="$emit('blur')"
    @focus="$emit('focus')"
  >
    <template #suffix>
      <button
        v-if="passwordToggle"
        type="button"
        class="rox-input__toggle"
        :aria-label="maskLabel"
        @click="toggleMask"
      >{{ masked ? '👁️' : '🙈' }}</button>

      <button
        v-if="clearable && String(display).length"
        type="button"
        class="rox-input__clear"
        @click="clear"
        aria-label="Clear"
      >✖</button>

      <slot name="suffix" />
    </template>
  </RoxInputCore>
  <p v-if="errors.length" class="rox-input__error">{{ errors[0] }}</p>
</template>

<script setup>
import { ref, computed } from 'vue'
import RoxInputCore from './RoxInputCore.vue'
import { usePasswordMask } from './plugins/usePasswordMask'
import { useFormatter } from './plugins/useFormatter'
import { useValidation } from './plugins/useValidation'

const props = defineProps({
  modelValue: { type: [String, Number], default: '' },
  id: { type: String, default: undefined },
  name: { type: String, default: undefined },
  type: { type: String, default: 'text' },
  disabled: { type: Boolean, default: false },
  readonly: { type: Boolean, default: false },
  ariaDescribedby: { type: String, default: undefined },

  // Plugin toggles / configs
  passwordToggle: { type: Boolean, default: false },
  clearable: { type: Boolean, default: false },

  // Formatter
  format: { type: Function, default: null },
  parse: { type: Function, default: null },

  // Validation
  rules: { type: Array, default: () => [] },
})
const emit = defineEmits(['update:modelValue', 'blur', 'focus'])

// 1) Masking
const { masked, inputType, toggle: toggleMask, label: maskLabel } = usePasswordMask(true)

// 2) Model + formatting
const mv = ref(props.modelValue)
const hasFormatter = computed(() => !!props.format && !!props.parse)
const { display, onInput: onFormattedInput } = useFormatter({
  modelValue: mv,
  emit,
  format: props.format ?? (x => x ?? ''),
  parse:  props.parse  ?? (x => x),
})
const onInput = (v) => hasFormatter.value ? onFormattedInput(v) : emit('update:modelValue', v)

// 3) Validation
const { errors, invalid } = useValidation({ modelValue: mv, rules: props.rules })

// 4) Clear
const clear = () => emit('update:modelValue', '')

// 5) Derived props
const resolvedType = computed(() => (props.passwordToggle ? inputType.value : props.type))
</script>

<style scoped>
.rox-input__toggle, .rox-input__clear { margin-left: .25rem; }
.rox-input__error { margin:.25rem 0 0; font-size:.875rem; color:#c00; }
</style>
```

---

### 2.4 Common Controls adapter (5‑prop minimal surface)

```vue
<template>
  <RoxInputCore
    :id="id"
    :name="name"
    :model-value="modelValue"
    :disabled="disabled"
    :readonly="readonly"
    :invalid="invalid"
    :aria-describedby="describedBy"
    @update:model-value="$emit('update:modelValue', $event)"
  />
</template>

<script setup>
import { RoxInputCore } from '@cnodigital/roxbury/src/components'

defineProps({
  modelValue: { type: [String, Number], default: '' },
  id: { type: String, default: undefined },
  name: { type: String, default: undefined },
  disabled: { type: Boolean, default: false },
  readonly: { type: Boolean, default: false },
  invalid: { type: Boolean, default: false },
  describedBy: { type: String, default: undefined },
})
defineEmits(['update:modelValue'])
</script>
```

---

## 3. Component‑Level Validation Gates (per audit row)

- **Bundle size:** Δ JS ≤ +2% and Δ CSS ≤ +2% (document exceptions).
- **API compatibility:** Adapter shims pass all unit tests.
- **Visual:** Playwright screenshots on pilot SPA routes approved.
- **A11y:** `axe-playwright` passes; no focus/ARIA regressions.
- **Perf:** Local Lighthouse smoke; production vitals unchanged within normal variance.

---

# Component Bundle Size Visualization Guide

## 1. Why
Ensures replacing Common Controls components with Roxbury ones doesn’t inflate bundles.

## 2. Setup for Roxbury (Vite Lib Mode)

Add analyzer dependency:
```bash
npm i -D rollup-plugin-visualizer
```

`vite.config.ts`
```ts
import { visualizer } from 'rollup-plugin-visualizer'

export default {
  build: {
    lib: { entry: 'src/index.ts', name: 'roxbury' },
    rollupOptions: {
      plugins: [visualizer({ filename: 'bundle-report.html', gzipSize: true })],
    },
  },
}
```

Run build:
```bash
npm run build
# open dist/bundle-report.html
```

---

## 3. Setup for Common Controls (No Lib Mode)

If Common Controls doesn’t use Vite’s `build.lib` config, create a temporary **analyzer entry** that bundles all SFCs once for analysis.

### Step 1 — Install analyzer
```bash
npm i -D vite @vitejs/plugin-vue rollup-plugin-visualizer
# or for Vue 2: npm i -D vite vite-plugin-vue2 rollup-plugin-visualizer
```

### Step 2 — Temporary config file `analyze.config.js`
```js
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue' // or: import vue from 'vite-plugin-vue2'
import { visualizer } from 'rollup-plugin-visualizer'
import path from 'path'

export default defineConfig({
  plugins: [vue(), visualizer({ filename: 'bundle-analysis.html', gzipSize: true })],
  build: {
    rollupOptions: {
      input: path.resolve(__dirname, 'src/components/index.js'),
      output: { manualChunks: undefined },
    },
    outDir: 'analyze-dist',
  },
})
```

> **Note:** Replace `@vitejs/plugin-vue` with `vite-plugin-vue2` if Common Controls is on Vue 2.

### Step 3 — Run analyzer
```bash
npx vite build --config analyze.config.js
```

Open `analyze-dist/bundle-analysis.html` for an interactive treemap of all component sizes.

### Step 4 — Compare Reports (pre/post)
Commit the HTML reports:
```
/reports/
  ├── cc-input-pre.html
  └── cc-input-post.html
```

Use these as artifacts in Azure DevOps and reference them from the PR checklist.

---

# Summary

- Roxbury owns the UI; Common Controls wraps logic.
- All shared components migrate to Roxbury via Core + Plugins architecture.
- Adoption gates: size, compatibility, tests, accessibility, visual parity, web vitals.
- Component-level audit and analyzer guard changes.
- Layout/Header/Footer pilot validates process.
- CI analyzer artifacts make size deltas objective and reviewable.

---

*End of Document*
