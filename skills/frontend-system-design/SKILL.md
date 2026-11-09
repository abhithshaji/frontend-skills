---
name: frontend-system-design
description: Architect, implement, review, and optimize production React applications using frontend system design principles. Use when designing new React features, choosing rendering strategies (CSR, SSR, SSG, ISR, RSC, PPR), managing state and caching (TanStack Query, normalization, optimistic updates, client schema drift), building real-time or offline-first systems (WebSockets, SSE, NDJSON streaming, OT vs CRDT, Service Workers, IndexedDB), optimizing large lists and heavy data (virtualization, cursor pagination, infinite scroll, LexoRank drag-and-drop reordering, Web Workers), improving bundle size and Core Web Vitals (code splitting, lazy loading, tree shaking, preload/prefetch, LCP, CLS, INP), or enforcing accessibility (WCAG 2.2 AA), security, and resilience.
---
# Frontend System Design

A consolidated architectural playbook and implementation guide for designing scalable, resilient, accessible, and high-performance [React](https://react.dev/reference/react) applications. Use this skill alongside component composition and micro-optimization skills (`vercel-react-best-practices`, `vercel-composition-patterns`, `web-design-guidelines`) to make system-level decisions across rendering, data synchronization, client storage, concurrency, and user experience.

## When to Use

- Designing the system architecture for a new React feature, route, or full-scale web application
- Selecting or migrating between rendering strategies (CSR, [SSR](https://nextjs.org/docs/pages/building-your-application/rendering/server-side-rendering), SSG, ISR, [React Server Components](https://nextjs.org/docs/app/getting-started/server-and-client-components), [Partial Prerendering](https://nextjs.org/docs/15/app/getting-started/partial-prerendering))
- Implementing data fetching, multi-layer caching, normalized relational state, or [optimistic UI updates](https://react.dev/reference/react/useOptimistic)
- Building real-time collaborative features, byte-level AI token streaming UIs, or offline-first sync engines
- Scaling data-heavy views with [DOM virtualization](https://tanstack.com/virtual/latest/docs/introduction), cursor-based infinite scroll, O(1) drag-and-drop reordering, or [Web Workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API)
- Auditing and fixing bundle bloat, [Core Web Vitals](https://web.dev/articles/vitals) regressions ([LCP](https://web.dev/articles/lcp), [CLS](https://web.dev/articles/cls), [INP](https://web.dev/articles/inp)), or [WCAG 2.2](https://www.w3.org/TR/WCAG22/) accessibility issues
- Hardening client-side storage against schema drift, XSS/CSRF vulnerabilities, and network failures

## Steps: System Design Workflow

When asked to design, implement, or review a React system, follow this 6-step RADIO-aligned workflow:

1. **Clarify Requirements and Scope**: Identify functional user flows, data volume (50 vs 50,000 items), update frequency (static vs real-time), SEO needs, offline requirements, and target devices.
2. **Select Rendering and Routing Architecture**: Use the Rendering Strategy Selection tree below to choose CSR, SSR, SSG, ISR, RSC, or PPR.
3. **Design State Taxonomy and Data Model**: Partition state strictly into Server Cache, URL State, Local UI State, Shared Client Store (normalized if relational), and Ephemeral/Form State.
4. **Architect Network, Sync, and Storage**: Define pagination strategy (Cursor vs Offset), mutation handling (Optimistic UI with rollback), real-time transport ([SSE](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events) vs [WebSockets](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API) vs [CRDTs](https://docs.yjs.dev/)), and versioned client storage.
5. **Optimize Runtime and Bundle Performance**: Apply code splitting, tree-shakable imports, resource hints ([`preload`](https://react.dev/reference/react-dom/preload), [`preconnect`](https://react.dev/reference/react-dom/preconnect), [`preinit`](https://react.dev/reference/react-dom/preinit)), list virtualization, and Web Worker offloading to hit [Core Web Vitals thresholds](https://web.dev/articles/defining-core-web-vitals-thresholds).
6. **Enforce Accessibility, Security, and Resilience**: Verify [WCAG 2.2 AA](https://www.w3.org/WAI/standards-guidelines/wcag/new-in-22/) keyboard and screen-reader flows, wrap feature boundaries in Error Boundaries, sanitize untrusted inputs with [DOMPurify](https://github.com/cure53/dompurify), and attach telemetry.

## Master Architectural Decision Trees

### 1. Rendering Strategy Selection

```text
Is the page content public and indexable?
├── No (Behind Auth: Dashboards, Internal Tools, Editors)
│   └── Needs heavy server-only data joins or zero-bundle static shells?
│       ├── Yes -> Hybrid RSC + Client Islands (or CSR with preloaded shell)
│       └── No  -> CSR (Vite SPA) + TanStack Query + Route Code Splitting
└── Yes (Marketing, E-commerce, Docs, Social Feeds)
    └── Does the page combine static chrome with personalized/live regions?
        ├── Yes -> Partial Prerendering (PPR) + Streaming Suspense
        └── No  -> How frequently does the underlying data change?
            ├── Per-request / Real-time -> SSR + React Server Components (RSC)
            ├── Seconds to minutes      -> Incremental Static Regeneration (ISR)
            └── On deploy only          -> Static Site Generation (SSG)
```

### 2. Performance Bottleneck Triage

```text
Where is the primary bottleneck?
├── Network / Initial Load Slow (High LCP / FCP)
│   ├── Large JS bundle   -> Route & Component Code Splitting + Tree Shaking (No Barrel Files)
│   ├── Late asset fetch  -> react-dom preload() / preconnect() + fetchPriority="high" on LCP
│   └── Waterfall requests -> Parallelize fetches / Prefetch on hover or route loader
├── DOM / Rendering Lag (Low FPS on Scroll or Large Lists)
│   ├── > 100 complex rows -> DOM Virtualization (@tanstack/react-virtual or react-window)
│   ├── Massive dataset    -> Cursor Pagination + Infinite Scroll (Never Offset on Infinite Scroll)
│   └── Re-render cascades -> Normalize state (O(1) entity lookup) + Colocate state
└── Main Thread Blocked (High INP > 200ms / Frozen Input)
    ├── Heavy CPU compute  -> Offload to Web Worker with requestId race protection
    └── Expensive React UI -> Wrap non-urgent updates in startTransition / useDeferredValue
```

### 3. Real-Time and Collaborative Sync Selection

```text
What is the direction and conflict model of live data?
├── Server-to-Client Only (AI token streaming, live feeds, notifications, tickers)
│   └── Use Server-Sent Events (SSE) or Fetch ReadableStream (NDJSON)
├── Bidirectional Low-Latency (Chat, multiplayer presence, live rooms)
│   └── Use WebSockets (backed by Redis Pub/Sub across multiple servers)
└── Concurrent Multi-User Document Editing
    ├── Centralized server authority, online-primary -> Operational Transformation (OT)
    └── Offline-first, P2P capable, decentralized    -> CRDT (Yjs / Automerge)
```

## Core Architecture Rules by Domain

### Domain 1: State, Caching, Normalization, and Schema Safety

Read [references/data-state-caching.md](references/data-state-caching.md) when designing data fetching, state stores, mutations, or local persistence.

- **Server Truth Principle**: Treat frontend state as a temporal snapshot; delegate remote server state to [TanStack Query](https://tanstack.com/query/latest/docs/framework/react/guides/optimistic-updates) rather than copying API responses into global client stores.
- **Cache Freshness vs Retention**: Configure `staleTime` based on data volatility and blast radius (how long cached data is reused without refetching) and `gcTime` based on memory retention of inactive queries.
- **State Normalization**: Flatten deeply nested relational API payloads into `{ entities: { byId }, ids: [] }` lookup tables using [Zustand](https://github.com/pmndrs/zustand) or [Redux Toolkit `createEntityAdapter`](https://redux.js.org/toolkit/api/createEntityAdapter) so updates are O(1) and components subscribe to individual entity IDs without cascading re-renders.
- **React 19 Optimistic Updates**: Always invoke [`useOptimistic`](https://react.dev/reference/react/useOptimistic) updaters inside `startTransition` or Form Actions, or use TanStack Query `onMutate` with query cancellation, a saved snapshot, and `onError` rollback.
- **Client Schema Drift Protection**: Never read raw [`localStorage`](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage) or [`IndexedDB`](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API) without `try/catch` and runtime validation (e.g., Zod). Use **version tagging** (`{ version: 2, data }`) for forward compatibility and **key versioning** (`app:filters:v2`) to prevent rollback crashes when an older release reads newer storage.

### Domain 2: Real-Time, Streaming UI, Offline-First, and CRDTs

Read [references/realtime-streaming-offline.md](references/realtime-streaming-offline.md) when building live updates, AI chat streams, collaborative editors, or offline support.

- **Streaming Byte Pipelines**: For AI responses or chunked endpoints, read `response.body.getReader()` via the [`ReadableStream` API](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream) with `new TextDecoder('utf-8', { stream: true })`, maintain a trailing incomplete-line buffer before `JSON.parse` on NDJSON lines, and wire an `AbortController` for cleanup.
- **WebSocket Resilience & Scaling**: Implement exponential backoff with random jitter on disconnect, push incoming socket deltas directly into the query cache via `queryClient.setQueryData`, and use a message broker (Redis Pub/Sub) when scaling across multiple WebSocket servers.
- **Offline-First Architecture**: Pair [Service Worker](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API) asset caching (`Cache-First` for hashed bundles, `Stale-While-Revalidate` for shells) with an [IndexedDB](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API) mutation outbox queue for offline writes. Do not rely solely on `navigator.onLine`; verify actual reachability.
- **Collaborative Conflict Resolution**: Avoid Last-Write-Wins (LWW) for shared text or rich state. Use **Operational Transformation (OT)** when a central server coordinates positional index transformations, or **CRDTs** ([Yjs](https://docs.yjs.dev/)) with globally unique immutable node IDs for decentralized, offline-capable merging.

### Domain 3: Rendering, Bundle Optimization, and Core Web Vitals

Read [references/rendering-bundle-performance.md](references/rendering-bundle-performance.md) when configuring SSR/RSC/PPR, code splitting, resource hints, or fixing LCP/CLS/INP.

- **Code Splitting and Chunk Recovery**: Split routes and heavy conditional components (modals, charts, editors) with `React.lazy` and `<Suspense>`. Always wrap lazy boundaries in an Error Boundary that catches `ChunkLoadError` after new deployments.
- **Tree Shaking Discipline**: Use ES module named imports, avoid barrel files (`index.ts` re-exporting entire directories), mark `"sideEffects": false` in `package.json`, and dynamically `await import()` heavy event-driven libraries (such as PDF exporters).
- **Resource Hints**: Use `react-dom` functions ([`preconnect`](https://react.dev/reference/react-dom/preconnect), [`preload`](https://react.dev/reference/react-dom/preload), [`preinit`](https://react.dev/reference/react-dom/preinit)) and prefetch route chunks plus query data on `onMouseEnter` or `onFocus` to eliminate navigation latency.
- **Core Web Vitals Budgets**:
  - **[LCP](https://web.dev/articles/lcp) (<= 2.5s)**: Never lazy-load the hero image; use `loading="eager"`, `fetchPriority="high"`, and preload critical assets.
  - **[CLS](https://web.dev/articles/cls) (<= 0.1)**: Reserve explicit `width`, `height`, or CSS `aspect-ratio` on all media, skeletons, and dynamic slots.
  - **[INP](https://web.dev/articles/inp) (<= 200ms)**: Keep main-thread tasks under 50ms; wrap expensive state transitions in `startTransition` or [`useDeferredValue`](https://react.dev/reference/react/useDeferredValue).

### Domain 4: Large Lists, Pagination, Reordering, and Web Workers

Read [references/scale-lists-workers.md](references/scale-lists-workers.md) when handling thousands of rows, drag-and-drop boards, pagination, or CPU-intensive tasks.

- **DOM Virtualization**: Render only viewport-visible items plus a small `overscan` (3 to 5 items) using [`@tanstack/react-virtual`](https://tanstack.com/virtual/latest/docs/introduction) (supporting dynamic row heights and `anchorTo: 'end'` for chat/log streams) or `react-window`.
- **Pagination Architecture**: Use **Offset Pagination** only for bounded tables requiring random page jumps (`Page 1, 2, 10`). Always back **Infinite Scroll** and deep feeds with **Cursor (Keyset) Pagination** (`WHERE id > cursor`) to guarantee O(1) index seeks and prevent duplicate or skipped rows on live insertions.
- **Reordering at Scale**: Never re-index sequential integers (`0, 1, 2...`) across thousands of items on drag-and-drop. Use **LexoRank** (lexicographical string midpoints) or **Fractional Indexing** for O(1) single-item updates, paired with optimistic local mutation, debounced API persistence, and keyboard-accessible move controls per [WCAG 2.2 Dragging Movements (2.5.7)](https://www.w3.org/WAI/standards-guidelines/wcag/new-in-22/).
- **Web Worker Offloading**: Debouncing an input only delays when a heavy task starts; it does not stop a 400ms synchronous filter over 50,000 items from freezing the main thread. Offload CPU-heavy parsing, filtering, and sorting to a [Web Worker](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API), send the dataset once, pass lightweight query payloads with a monotonic `requestId` to discard stale responses, and use Transferable `ArrayBuffer` objects when applicable.

### Domain 5: Accessibility (a11y), Security, Resilience, and Observability

Read [references/a11y-security-resilience-ops.md](references/a11y-security-resilience-ops.md) when building UI components, forms, menus, modals, auth flows, error handling, or i18n.

- **Golden Rule of ARIA and Focus**: Prefer native semantic HTML (`<button>`, `<nav>`, `<main>`, `<dialog>`) over custom `<div>` roles. Use `tabIndex={0}` to include a custom widget in the natural tab sequence and `tabIndex={-1}` for programmatic `.focus()` only. **Never use positive `tabIndex` values (`tabIndex >= 1`)**, as they hijack natural DOM focus order and break keyboard navigation.
- **Interactive Widget Patterns**: Link form inputs, helper text, and `role="alert"` error messages using React `useId()`, `aria-invalid`, and `aria-describedby`. Implement `ArrowUp`/`ArrowDown`/`Home`/`End`/`Escape` keyboard handlers and focus restoration on custom menus and modals, and ensure interactive targets meet the [24x24 CSS pixel minimum (WCAG 2.5.8)](https://www.w3.org/WAI/WCAG22/Understanding/target-size-minimum.html).
- **SPA Route Announcements**: On client-side route transitions, update `document.title` and move focus programmatically (`tabIndex={-1}`) or announce the change via an `aria-live` region so screen reader users know navigation occurred.
- **Security and Resilience**: Sanitize any HTML passed to `dangerouslySetInnerHTML` with [DOMPurify](https://github.com/cure53/dompurify), avoid storing sensitive tokens in `localStorage` (prefer `HttpOnly` cookies + in-memory access tokens), isolate failures with granular Error Boundaries, and track real-user CWV metrics in production.

## Gotchas

- **Positive `tabIndex` values (`tabIndex={1}` or higher)**: Reject immediately; restructure DOM order and use `tabIndex={0}` or `tabIndex={-1}`.
- **`useOptimistic` outside a Transition**: Calling `addOptimistic()` outside `startTransition` or a Form Action causes state desynchronization and React runtime warnings.
- **Missing `{ stream: true }` in `TextDecoder.decode()`**: Decoding chunked `Uint8Array` streams without `{ stream: true }` corrupts multi-byte UTF-8 characters split across network packets.
- **Infinite Scroll over Offset Pagination**: Causes O(N) database slowdown at depth and duplicate React keys when new records are inserted at the top.
- **Debouncing as a substitute for Web Workers**: A 300ms debounce still freezes the main thread for 400ms once the callback fires.
- **Trusting `localStorage` without schema versioning**: Unversioned JSON blobs crash the app on forward deploys or rollbacks.
- **Barrel files (`export * from './components'`) in large apps**: Defeats tree shaking and inflates cold-start and chunk sizes.

## Reference Playbooks

Load the relevant reference file when implementing or reviewing specific system design domains:

- [references/data-state-caching.md](references/data-state-caching.md) - State taxonomy, multi-tier caching, JSON normalization, optimistic updates, and client schema drift.
- [references/realtime-streaming-offline.md](references/realtime-streaming-offline.md) - Polling/SSE/WebSockets, Redis Pub/Sub scaling, NDJSON streaming UI, OT vs CRDT, and offline-first sync.
- [references/rendering-bundle-performance.md](references/rendering-bundle-performance.md) - CSR/SSR/SSG/ISR/RSC/PPR, hydration, code splitting, lazy loading, tree shaking, preload/prefetch, and Core Web Vitals.
- [references/scale-lists-workers.md](references/scale-lists-workers.md) - List virtualization, offset vs cursor pagination, infinite scroll, LexoRank list reordering, and Web Workers.
- [references/a11y-security-resilience-ops.md](references/a11y-security-resilience-ops.md) - WCAG 2.2 AA accessibility, keyboard/screen-reader patterns, error boundaries, security, observability, and i18n.
