# Frontend System Design Skill

A consolidated architectural playbook and reference guide for designing scalable, resilient, accessible, and high-performance frontend and React applications.

## Overview

This skill provides structured guidance and architectural decision trees for frontend engineers and AI agents. It covers high-level system design workflows (RADIO framework), rendering strategies (CSR, SSR, SSG, ISR, RSC, PPR), state taxonomy and multi-layer caching, real-time streaming and offline-first synchronization, high-scale DOM virtualization and Web Workers, bundle optimization and Core Web Vitals, as well as production accessibility (WCAG 2.2 AA), security, and resilience.

## Structure

```
frontend-system-design/
├── SKILL.md                  # Main skill entry point & master architectural decision trees
├── README.md                 # Skill documentation and overview
└── references/               # Deep-dive architectural playbooks with production implementations
    ├── data-state-caching.md           # State taxonomy, multi-tier caching, JSON normalization, optimistic UI, schema drift
    ├── realtime-streaming-offline.md   # Polling/SSE/WebSockets, Redis Pub/Sub, NDJSON streaming, OT vs CRDT, Service Workers
    ├── rendering-bundle-performance.md # CSR/SSR/SSG/ISR/RSC/PPR, code splitting, tree shaking, resource hints, Core Web Vitals
    ├── scale-lists-workers.md          # DOM virtualization, offset vs cursor pagination, LexoRank reordering, Web Workers
    └── a11y-security-resilience-ops.md # WCAG 2.2 AA, keyboard & focus patterns, Error Boundaries, security, telemetry
```

## How to Use This Skill

### For AI Agents
1. Read [`SKILL.md`](./SKILL.md) to inspect the master architectural decision trees (Rendering Selection, Performance Bottlenecks, Real-Time Transport) and follow the 6-step system design workflow.
2. Read the specific reference guide in [`references/`](./references) before architecting, implementing, or reviewing system-level components.
3. Enforce the core architectural rules and avoid documented gotchas (e.g., forbidding positive `tabIndex`, preventing offset pagination on infinite feeds, protecting client storage against schema drift, and correctly buffering streamed UTF-8 chunks).

### For Developers
- **State & Data Caching**: Refer to [`data-state-caching.md`](./references/data-state-caching.md) when partitioning state into the 5-layer taxonomy, configuring TanStack Query cache policies, normalizing relational JSON payloads, executing React 19 optimistic updates, or versioning `localStorage` and `IndexedDB` against schema drift.
- **Real-Time, Streaming & Offline**: Consult [`realtime-streaming-offline.md`](./references/realtime-streaming-offline.md) when evaluating SSE vs WebSockets, scaling sockets horizontally with Redis Pub/Sub, parsing chunked NDJSON/AI streams with `ReadableStream`, resolving collaborative conflicts (OT vs CRDT/Yjs), or setting up Service Worker offline outboxes.
- **Rendering & Core Web Vitals**: Check [`rendering-bundle-performance.md`](./references/rendering-bundle-performance.md) when choosing between CSR, SSR, SSG, ISR, RSC, and PPR, eliminating hydration mismatches, optimizing bundle size and tree shaking (no barrel files), adding resource hints (`preload`, `preconnect`), and hitting LCP, CLS, and INP thresholds.
- **Large Lists, Pagination & Workers**: Refer to [`scale-lists-workers.md`](./references/scale-lists-workers.md) when implementing windowed virtualization with `@tanstack/react-virtual`, transitioning from offset to cursor pagination for infinite feeds, implementing O(1) drag-and-drop reordering with LexoRank, or offloading CPU bottlenecks to Web Workers.
- **Accessibility, Security & Resilience**: Inspect [`a11y-security-resilience-ops.md`](./references/a11y-security-resilience-ops.md) when auditing WCAG 2.2 AA compliance, implementing semantic roving tabindex, managing SPA route announcements, preventing XSS with DOMPurify, securing tokens, and establishing granular Error Boundaries.

## Core Guidelines

- **Systematic Decision-Making**: Anchor technical proposals in the 6-step RADIO framework (Requirements, Architecture/Rendering, Data/State, Network/Sync, Optimization, and Accessibility/Security).
- **Server Truth & Normalized State**: Treat client stores as temporary snapshots; delegate server state to dedicated query caches and normalize relational entities to avoid cascading re-renders.
- **Non-Blocking Main Thread**: Keep main-thread tasks under 50ms. Use `startTransition` or `useDeferredValue` for heavy rendering, DOM virtualization for large lists, and Web Workers for CPU-bound computations.
- **Resilient & Offline-First Design**: Protect persistent stores with schema versioning (`{ version, data }`), buffer streamed network responses cleanly, and back offline actions with reliable mutation queues.
- **Accessibility & Security by Default**: Prefer native semantic HTML over custom ARIA, never use positive `tabIndex`, ensure minimum touch targets (24x24px), sanitize untrusted content, and isolate failures with localized Error Boundaries.

