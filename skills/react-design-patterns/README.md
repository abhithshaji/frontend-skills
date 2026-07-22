# React Design Patterns Skill

A consolidated playbook and reference guide of React design patterns for building, refactoring, and reviewing high-quality React components.

## Overview

This skill provides structured guidance and reference patterns for React developers and AI agents. It covers modern React standards (including React 19), component composition, state management, asynchronous UI, performance optimization, and architectural best practices.

## Structure

```
react-design-patterns/
├── SKILL.md                  # Main skill entry point & pattern decision matrix
├── README.md                 # Skill documentation and overview
└── references/               # Detailed pattern references with code implementations
    ├── composition-patterns.md # Compound Components, Slot Pattern, Container-Presenter, Render Props/HOC
    ├── state-patterns.md       # Custom Hooks, Provider (Context), State Reducer, Controlled/Uncontrolled, Pub-Sub
    ├── async-patterns.md       # Suspense + use(), Error Boundaries, Optimistic UI, Facade Hooks, Strategy Hooks
    ├── performance-patterns.md # Memoization, Derived State, Debounce/Throttle, Lazy Loading, Virtualization, Concurrent UI
    └── crud-example.md         # Full worked CRUD example combining multiple design patterns
```

## How to Use This Skill

### For AI Agents   
1. Read [`SKILL.md`](./SKILL.md) to inspect the pattern decision table and identify appropriate patterns for a given task.
2. Read the specific reference guide in [`references/`](./references) before implementing or refactoring React components.
3. Enforce the core rules outlined in `SKILL.md` (e.g., Rules of Hooks, functional state updates, derived state rules).

### For Developers
- **Component Architecture**: Refer to [`composition-patterns.md`](./references/composition-patterns.md) when resolving prop proliferation ("prop soup") or separating presentational layout from business logic.
- **State Management**: Consult [`state-patterns.md`](./references/state-patterns.md) when sharing logic across components, managing complex state transitions, or avoiding prop drilling.
- **Async Operations & Feedback**: Refer to [`async-patterns.md`](./references/async-patterns.md) for data fetching, suspense boundaries, error handling, and instant UI updates.
- **Performance Optimization**: Check [`performance-patterns.md`](./references/performance-patterns.md) for profiling re-renders, list virtualization, debouncing high-frequency events, and non-blocking concurrent rendering.
- **Comprehensive Example**: Inspect [`crud-example.md`](./references/crud-example.md) for an end-to-end form and CRUD workflow composing multiple patterns.

## Core Guidelines

- **Simplicity First**: Prefer the simplest pattern that solves the problem. Patterns are tools, not goals.
- **Hooks-First Logic**: Prefer custom hooks over legacy HOC or Render Props patterns in modern React codebases.
- **React 19 Standards**: Utilize React 19 patterns such as `<Context value={...}>`, `use()`, `useTransition()`, and optimistic updates where applicable.
- **Immutability & Derived State**: Compute state dynamically during render whenever possible instead of duplicating state.
