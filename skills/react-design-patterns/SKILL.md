---
name: react-design-patterns
description: React design patterns reference for building, refactoring, or reviewing React components. Use when writing React code that involves component composition (compound components, slots, container-presenter), state management (custom hooks, providers, reducers, pub-sub), async UI (Suspense, error boundaries, optimistic updates), or performance optimization (memoization, virtualization, debounce, lazy loading, concurrent rendering). Also use when a component has too many props, duplicated logic, mixed data/UI concerns, unnecessary re-renders, or waterfall loading states.
---

# React Design Patterns

A consolidated playbook of React design patterns. Apply these when writing new components, refactoring existing ones, or reviewing React code. Prefer the smallest pattern that solves the problem — patterns are tools, not goals.

## How to use this skill

1. Identify the problem symptom in the table below.
2. Open the matching reference file for the full explanation, working code, and pitfalls.
3. Follow the "Core rules" at the bottom of this file in ALL React code, regardless of pattern.

## Pattern decision table

| Symptom / Need | Pattern | Reference |
|---|---|---|
| Component has too many props ("prop soup"), rigid layout variations | Compound Components | [composition-patterns.md](references/composition-patterns.md) |
| Consumer needs to inject UI into fixed regions of a layout | Slot Pattern | [composition-patterns.md](references/composition-patterns.md) |
| Component mixes data fetching + rendering; hard to test | Container-Presenter | [composition-patterns.md](references/composition-patterns.md) |
| Sharing render logic (legacy codebases; prefer hooks in new code) | Render Props / HOC | [composition-patterns.md](references/composition-patterns.md) |
| Same stateful logic duplicated across components | Custom Hooks | [state-patterns.md](references/state-patterns.md) |
| Prop drilling through many layers | Provider (Context) | [state-patterns.md](references/state-patterns.md) |
| Complex state transitions; consumer needs to customize state logic | State Reducer | [state-patterns.md](references/state-patterns.md) |
| Form inputs: who owns the value? | Controlled / Uncontrolled | [state-patterns.md](references/state-patterns.md) |
| Decoupled cross-component / cross-tab / micro-frontend events | Pub-Sub & Observer | [state-patterns.md](references/state-patterns.md) |
| Loading waterfalls, fetch-on-render, spinner soup | Suspense + `use()` | [async-patterns.md](references/async-patterns.md) |
| One crashing widget takes down the whole app | Error Boundary | [async-patterns.md](references/async-patterns.md) |
| UI should update instantly before the server confirms (likes, cart) | Optimistic UI | [async-patterns.md](references/async-patterns.md) |
| Component juggles many hooks/flags; messy consumption | Facade Hook | [async-patterns.md](references/async-patterns.md) |
| Swappable interchangeable behaviors (auth methods, payment providers) | Strategy + Hook Factory | [async-patterns.md](references/async-patterns.md) |
| Unnecessary re-renders of expensive children | Memoization (`memo`, `useMemo`, `useCallback`) | [performance-patterns.md](references/performance-patterns.md) |
| State that can be computed from other state/props | Derived State (don't store it) | [performance-patterns.md](references/performance-patterns.md) |
| Rapid-fire events (search input, scroll, resize) | Debounce & Throttle | [performance-patterns.md](references/performance-patterns.md) |
| Large bundle; routes/components not needed upfront | Lazy Loading (`React.lazy`) | [performance-patterns.md](references/performance-patterns.md) |
| Rendering 1,000+ list rows | Virtualization | [performance-patterns.md](references/performance-patterns.md) |
| Typing/interaction feels janky during heavy renders | `useTransition` / `useDeferredValue` | [performance-patterns.md](references/performance-patterns.md) |
| Full worked example: form + CRUD composing several patterns | CRUD Form Example | [crud-example.md](references/crud-example.md) |

## Choosing between similar patterns

- **Compound Components vs Slot Pattern**: compound components when children share implicit state (tabs, accordion, form fields via context). Slots when you only need layout placement with no shared state.
- **Custom Hook vs HOC vs Render Props**: always prefer a custom hook in new code. HOC/render props only when matching an existing legacy codebase style.
- **Context vs Pub-Sub**: reach for props first, then Context. Pub-sub only when Context can't work: cross-tab sync, micro-frontend communication, decoupled analytics/toast systems.
- **useState vs useReducer**: `useReducer` when the next state depends on complex transitions or multiple sub-values change together. The full *State Reducer pattern* (accepting a `reducer` prop) only for reusable library-style components where consumers must override behavior.
- **useTransition vs useDeferredValue**: `useTransition` when you control the state update (you call `set`). `useDeferredValue` when you only receive the value (e.g. as a prop).
- **Memoization**: measure first (React DevTools Profiler or `react-scan`). Don't sprinkle `useMemo`/`useCallback` everywhere; if the project has React Compiler enabled, manual memoization is mostly unnecessary.

## Core rules (always apply)

1. **Rules of Hooks**: only call hooks at the top level of React function components or other hooks — never in loops, conditions, nested functions, or plain JS functions. Hook names start with `use`.
2. **Never store derived state.** If a value can be computed from existing state/props, compute it during render (wrap in `useMemo` only if expensive).
3. **Functional state updates** when the next state depends on the previous: `setItems(prev => [...prev, item])` — avoids stale closures.
4. **Never mutate state** — reducers and updaters must return new objects/arrays.
5. **Effects need cleanup**: abort fetches (`AbortController`), clear timers, unsubscribe listeners in the effect's return function.
6. **Error boundaries around independent widgets**, never only one around the app root.
7. **Keep presentational components dumb** — data fetching and business logic live in hooks/containers.
8. In React 19+ prefer `use(Context)` and `<Context value={...}>` over `useContext` / `<Context.Provider>` where the codebase targets React 19; match the existing codebase otherwise.
