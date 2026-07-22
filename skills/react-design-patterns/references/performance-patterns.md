# Performance Patterns

Memoization, Derived State, Debounce & Throttle, Lazy Loading, Virtualization, and Concurrent Rendering (`useTransition` / `useDeferredValue`).

## When does a component re-render?

1. Its **state** changes
2. Its **props** change
3. A **context** it consumes changes
4. Its **parent re-renders** (even with identical props — unless memoized)

**Measure before optimizing:** React DevTools Profiler, or the `react-scan` tool to highlight wasteful renders live. If the project uses the **React Compiler**, manual `memo`/`useMemo`/`useCallback` is mostly unnecessary — don't add it.

---

## Memoization

Cache a result for given inputs; recompute only when inputs change. Three tools:

| Tool | Caches | Use for |
|---|---|---|
| `memo(Component)` | the rendered output | skipping re-render when props are shallow-equal |
| `useMemo(fn, deps)` | the **return value** of `fn` | expensive computations; stable object/array references |
| `useCallback(fn, deps)` | the **function reference** itself | stable callbacks passed to memoized children / effect deps |

`useCallback(fn, deps)` ≡ `useMemo(() => fn, deps)`.

### The classic trap: memo defeated by a fresh function reference

```jsx
// Child.jsx
import { memo } from "react";
const Child = memo(function Child({ onClick }) {
  console.log("Child rendered");
  return <button onClick={onClick}>Click Me</button>;
});
```

```jsx
// Parent — ❌ Child re-renders on every count change anyway:
const handleClick = () => console.log("clicked");
// each parent render creates a NEW handleClick reference → memo sees "props changed"

// ✅ Fix — stable reference:
const handleClick = useCallback(() => console.log("clicked"), []);
```

`memo` only helps if **every** prop is referentially stable — one inline object/array/function prop defeats it. Same for context values and effect dependencies: memoize objects passed as `value={...}` or listed in deps.

### useMemo for expensive computation

```jsx
function Users({ list }) {
  // Without useMemo, a 10,000-item sort reruns on EVERY parent render
  const sorted = useMemo(
    () => [...list].sort((a, b) => a.localeCompare(b)),
    [list]
  );
  return sorted.map(user => <div key={user}>{user}</div>);
}
```

Note `[...list].sort()` — `.sort()` mutates; copy first.

**Don't** memoize cheap computations or every callback "just in case" — the bookkeeping has its own cost and clutters the code.

---

## Derived State — don't store it

Anything computable from existing state/props must be **computed during render**, never mirrored into another `useState`.

```jsx
// ❌ Redundant state + effect: extra render cycle, can go stale, more code
const Cart = ({ items }) => {
  const [total, setTotal] = useState(0);
  useEffect(() => {
    setTotal(items.reduce((acc, item) => acc + item.price, 0));
  }, [items]);
  return <h2>Total: {total}</h2>;
};

// ✅ Just compute it
function Cart({ items }) {
  const total = items.reduce((acc, item) => acc + item.price, 0);
  return <h2>Total: {total}</h2>;
}
```

If the derivation is expensive, wrap in `useMemo` — still not state. Red flag to hunt for in reviews: `useEffect` whose only job is `setState` from other state/props.

---

## Debounce & Throttle

- **Debounce** — wait until events *stop* for N ms, then act once. (Search-as-you-type: call the API after the user pauses.)
- **Throttle** — act at most once every N ms *while* events keep firing. (Scroll/resize/mousemove handlers.)

```jsx
// useDebounce.js — debounce a VALUE
import { useState, useEffect } from "react";

export function useDebounce(value, delay = 500) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const id = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(id); // each keystroke cancels the previous timer
  }, [value, delay]);

  return debouncedValue;
}
```

```jsx
// SearchBox.jsx
export default function SearchBox() {
  const [query, setQuery] = useState("");
  const debouncedQuery = useDebounce(query, 600);

  useEffect(() => {
    if (!debouncedQuery) return;
    // fetch(`/api/search?q=${debouncedQuery}`) — fires only after typing pauses
  }, [debouncedQuery]);

  return <input value={query} onChange={(e) => setQuery(e.target.value)} />;
}
```

```jsx
// useThrottle.js — emit a value at most once per `interval`
import { useState, useEffect, useRef } from "react";

export function useThrottle(value, interval = 300) {
  const [throttledValue, setThrottledValue] = useState(value);
  const lastExecuted = useRef(Date.now());

  useEffect(() => {
    const elapsed = Date.now() - lastExecuted.current;
    if (elapsed >= interval) {
      lastExecuted.current = Date.now();
      setThrottledValue(value);
    } else {
      // trailing update so the final value always lands
      const id = setTimeout(() => {
        lastExecuted.current = Date.now();
        setThrottledValue(value);
      }, interval - elapsed);
      return () => clearTimeout(id);
    }
  }, [value, interval]);

  return throttledValue;
}
```

```jsx
// ScrollTracker.jsx
export default function ScrollTracker() {
  const [scrollY, setScrollY] = useState(0);
  const throttledY = useThrottle(scrollY, 300);

  useEffect(() => {
    const handleScroll = () => setScrollY(window.scrollY);
    window.addEventListener("scroll", handleScroll, { passive: true });
    return () => window.removeEventListener("scroll", handleScroll);
  }, []);

  return <p>Scroll: {throttledY}</p>;
}
```

For debouncing/throttling **callbacks** (not values), or in production, `lodash.debounce` / `lodash.throttle` inside `useMemo` is battle-tested. Also compare `useDeferredValue` below — often the more React-native fix for expensive-render (not network) cases.

---

## Lazy Loading (Code Splitting)

Load component code on demand instead of in the initial bundle — smaller bundles, faster first paint.

```jsx
import React, { Suspense, useState } from "react";
import Light from "./Light";

// Dynamic import → separate chunk, fetched on first render of <Heavy />
const Heavy = React.lazy(() => import("./Heavy"));

export default function LazyDemo() {
  const [show, setShow] = useState(false);

  return (
    <div>
      <Light />
      <button onClick={() => setShow(s => !s)}>
        {show ? "Hide" : "Show"} Heavy
      </button>

      <Suspense fallback={<div>Loading heavy component…</div>}>
        {show && <Heavy />}
      </Suspense>
    </div>
  );
}
```

Requirements/notes:
- The lazy component must be a **default export** (or map it: `import("./m").then(m => ({ default: m.Heavy }))`).
- Always wrap in `<Suspense>` with a fallback.
- Best ROI: **route-level splitting** (each page a lazy chunk), heavy rarely-used components (charts, editors, modals).
- Pair with an Error Boundary — the chunk fetch can fail on flaky networks.

---

## Virtualization (Windowing)

Rendering 10,000 DOM rows kills performance. Virtualization renders **only the rows visible in the viewport** (plus a small overscan) and swaps them as the user scrolls.

Libraries: `react-window` (simple) or `@tanstack/react-virtual` (headless, flexible). Don't hand-roll.

```jsx
// react-window (v1 API)
import { FixedSizeList as List } from "react-window";

export default function VirtualList({ users, height = 600, itemHeight = 80 }) {
  return (
    <List
      height={height}
      itemCount={users.length}
      itemSize={itemHeight}
      width="100%"
      itemData={users}
      overscanCount={5} // extra rows above/below to reduce pop-in
    >
      {({ index, style, data }) => (
        // MUST apply `style` — it positions the row absolutely
        <div style={style}>{data[index].name}</div>
      )}
    </List>
  );
}
```

```jsx
// @tanstack/react-virtual — headless, supports dynamic row heights
import { useVirtualizer } from "@tanstack/react-virtual";
import { useRef } from "react";

export default function VirtualList({ users }) {
  const parentRef = useRef(null);
  const virtualizer = useVirtualizer({
    count: users.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 80,
    overscan: 5,
  });

  return (
    <div ref={parentRef} style={{ height: 600, overflow: "auto" }}>
      <div style={{ height: virtualizer.getTotalSize(), position: "relative" }}>
        {virtualizer.getVirtualItems().map(item => (
          <div
            key={item.key}
            style={{
              position: "absolute",
              top: 0,
              width: "100%",
              height: item.size,
              transform: `translateY(${item.start}px)`,
            }}
          >
            {users[item.index].name}
          </div>
        ))}
      </div>
    </div>
  );
}
```

**Use when:** lists/tables/grids beyond a few hundred rows, chat histories, log viewers. Note: check the installed `react-window` major version — v2 renamed the API; match what the project uses.

---

## Concurrent Rendering

Mark expensive updates as **non-urgent** so urgent ones (typing, clicking) stay instant. React can interrupt/abandon non-urgent renders.

### useTransition — you control the state update

```jsx
const [isPending, startTransition] = useTransition();
```

```jsx
export default function FilterDemo() {
  const [text, setText] = useState("");   // urgent — input echo
  const [query, setQuery] = useState(""); // non-urgent — heavy filtering
  const [isPending, startTransition] = useTransition();

  function handleChange(e) {
    const val = e.target.value;
    setText(val); // immediate: input stays responsive
    startTransition(() => {
      setQuery(val); // interruptible: triggers the expensive re-render
    });
  }

  return (
    <div>
      <input value={text} onChange={handleChange} />
      <span>{isPending ? "Updating results…" : "Idle"}</span>
      <HeavyList query={query} />
    </div>
  );
}
```

### useDeferredValue — you only receive the value

Same goal, different control point: `useTransition` wraps the *state update*; `useDeferredValue` wraps the *value* (useful when the state setter isn't yours, e.g. value comes from props).

```jsx
export default function DeferredDemo() {
  const [text, setText] = useState("");
  const deferredText = useDeferredValue(text); // lags behind during heavy work

  return (
    <div>
      <input value={text} onChange={(e) => setText(e.target.value)} />
      <span>{text === deferredText ? "Up-to-date" : "Updating…"}</span>
      <ProductsList filter={deferredText} />
    </div>
  );
}
```

For either to skip re-rendering with the old value, the heavy child should be memoized (`memo` and/or `useMemo` on the filtered list).

**These fix CPU-bound render jank, not slow networks.** For network waiting, see the Suspense pattern in [async-patterns.md](async-patterns.md). Transitions also integrate with Suspense: state updates inside `startTransition` keep showing the previous UI instead of flashing fallbacks.
