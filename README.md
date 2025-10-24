# palm-tree

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

(omitted for brevity)
