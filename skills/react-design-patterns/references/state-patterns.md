# State Patterns

Custom Hooks, Provider (Context), State Reducer, Controlled/Uncontrolled Components, and Pub-Sub/Observer.

---

## Custom Hooks Pattern

Hooks are plain functions that are aware of React capabilities (state, refs, lifecycle). Extract duplicated stateful logic into a custom hook; components stay declarative.

### Rules of Hooks (non-negotiable)

1. **Only call hooks at the top level** — never inside loops, conditions, or nested functions. React matches hooks to their internal state by call order on every render; breaking the order corrupts state.
2. **Only call hooks from React function components or other hooks** — never plain JS functions.
3. **Names start with `use`** — this is how linters and React identify them.

### Examples

```jsx
// useToggle — the smallest useful hook
import { useState, useCallback } from "react";

export function useToggle(initial = false) {
  const [value, setValue] = useState(initial);
  const toggle = useCallback(() => setValue(prev => !prev), []);
  return [value, toggle];
}
```

```jsx
// useFetch — with abort + race-condition safety
import { useState, useEffect } from "react";

export function useFetch(url) {
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    if (!url) return;
    const controller = new AbortController();

    (async () => {
      setLoading(true);
      setError(null);
      try {
        const response = await fetch(url, { signal: controller.signal });
        if (!response.ok) throw new Error(`HTTP ${response.status}`);
        setData(await response.json());
      } catch (err) {
        // Ignore aborts — they happen on unmount or when url changes mid-flight
        if (err.name !== "AbortError") setError(err.message);
      } finally {
        setLoading(false);
      }
    })();

    // Cleanup prevents: state updates after unmount AND stale responses
    // overwriting newer ones when url changes quickly
    return () => controller.abort();
  }, [url]);

  return { data, error, loading };
}
```

**Note:** for production data fetching prefer TanStack Query / SWR (caching, dedupe, revalidation). `useFetch` is fine for simple cases and for understanding the pattern.

---

## Provider Pattern

Provide data to a component subtree without prop drilling, via the Context API.

**Steps:** ① create the context → ② create a provider component → ③ wrap the tree → ④ expose a custom hook.

```jsx
// context.js
import { createContext } from "react";
export const ThemeContext = createContext(null);
```

```jsx
// ThemeProvider.jsx
import { useState, useMemo, useCallback } from "react";
import { ThemeContext } from "./context";

export function ThemeProvider({ children }) {
  const [dark, setDark] = useState(false);

  const toggleTheme = useCallback(() => {
    setDark(prev => !prev);
    document.body.classList.toggle("dark");
  }, []);

  // Memoize the value object — otherwise every provider render creates a new
  // object reference and re-renders EVERY consumer
  const value = useMemo(() => ({ dark, toggleTheme }), [dark, toggleTheme]);

  return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>;
  // React 19+: <ThemeContext value={value}>{children}</ThemeContext>
}
```

```jsx
// useTheme.js — always expose a hook, never raw useContext at call sites
import { useContext } from "react"; // React 19+: import { use }
import { ThemeContext } from "./context";

export function useTheme() {
  const ctx = useContext(ThemeContext); // React 19+: use(ThemeContext)
  if (!ctx) throw new Error("useTheme must be used within <ThemeProvider>");
  return ctx;
}
```

**Best practices:**
- The guard throw gives an actionable error instead of a cryptic null crash.
- Every consumer re-renders when the context value changes — split contexts by update frequency (e.g. `ThemeContext` vs `UserContext`), or split state and dispatch into two contexts.
- Context is for low-frequency, tree-wide data (theme, auth, locale). For high-frequency or complex global state, use a store (Zustand, Redux Toolkit, Jotai).

---

## State Reducer Pattern

Two levels to this:

### Level 1 — `useReducer` for complex local state

A **reducer** is a pure function `(state, action) => newState` that centralizes how state changes. **Dispatching** means calling `dispatch(action)` to trigger it.

```jsx
const [state, dispatch] = useReducer(reducerFn, initialState);
```

Use over `useState` when: multiple sub-values change together, the next state depends on transition logic, or you want state logic testable in isolation.

### Level 2 — the actual State Reducer *pattern* (inversion of control)

A reusable component accepts a `reducer` prop so **consumers can override or extend its state logic** without forking the component. Popularized by Kent C. Dodds (Downshift).

```jsx
// reducers/toggle-reducer.js
export function toggleReducer(state, action) {
  switch (action.type) {
    case "toggle":
      // Always return NEW objects — never mutate (state.clicks++ is a bug)
      return { on: !state.on, clicks: state.clicks + 1 };
    default:
      return state;
  }
}

// Consumer-supplied override: cap at 3 clicks
export function customToggleReducer(state, action) {
  switch (action.type) {
    case "toggle":
      if (state.clicks >= 3) return state;
      return { on: !state.on, clicks: state.clicks + 1 };
    default:
      return state;
  }
}
```

```jsx
// Toggle.jsx — default reducer, overridable via prop
import { useReducer } from "react";
import { toggleReducer } from "./reducers/toggle-reducer";

export default function Toggle({ reducer = toggleReducer, onToggle }) {
  const [state, dispatch] = useReducer(reducer, { on: false, clicks: 0 });
  const { on, clicks } = state;

  function handleToggle() {
    dispatch({ type: "toggle" });
    onToggle?.(!on);
  }

  return (
    <button onClick={handleToggle}>
      {on ? "ON" : "OFF"} (Clicks: {clicks})
    </button>
  );
}

// Usage
<Toggle onToggle={console.log} />
<Toggle reducer={customToggleReducer} onToggle={console.log} />
```

Scale it up by putting `{ state, dispatch }` into a Provider (`<FormProvider reducer={customFormReducer}>`) so any child can dispatch — combines State Reducer + Provider + Compound Components.

**Use when:** building reusable/library-style components (toggles, comboboxes, forms) where consumers need to customize behavior you can't predict. For app-internal components, a plain `useReducer` is enough.

---

## Controlled & Uncontrolled Components

Who owns a form input's value?

### Controlled — React state owns the value

```jsx
function SearchInput() {
  const [query, setQuery] = useState("");
  return <input value={query} onChange={(e) => setQuery(e.target.value)} />;
}
```

Every keystroke re-renders. Use when the UI must react to the value as it changes: live validation, filtering, enabling a submit button, formatting as-you-type.

### Uncontrolled — the DOM owns the value, read it via ref

```jsx
function SignupForm() {
  const emailRef = useRef(null);

  function handleSubmit(e) {
    e.preventDefault();
    console.log(emailRef.current.value); // read on demand
  }

  return (
    <form onSubmit={handleSubmit}>
      <input ref={emailRef} defaultValue="" type="email" />
      <button>Sign up</button>
    </form>
  );
}
```

Key facts about `useRef`: changing `.current` does **not** cause a re-render, and the value persists across re-renders. That's exactly why uncontrolled inputs are cheap — no render per keystroke.

**Choosing:** controlled for anything interactive/validated live; uncontrolled for simple submit-and-read forms, file inputs (`<input type="file">` is always uncontrolled), or perf-sensitive huge forms. Form libraries: React Hook Form is uncontrolled-first (fast); Formik is controlled.

**Component API design:** reusable inputs should support both — `value` (controlled) and `defaultValue` (uncontrolled) — like native inputs do. Never switch a mounted input between the two (React warns).

---

## Pub-Sub & Observer Pattern

- **Observer:** a *subject* keeps a list of observers and notifies them directly on events. One-to-many, tightly known relationship.
- **Pub-Sub:** publishers and subscribers never know each other; a **message broker (event bus)** sits in between. Many-to-many, fully decoupled.

### Event bus

```jsx
// lib/eventBus.js — Map of eventName -> Set of handlers
const listeners = new Map();

export const eventBus = {
  subscribe(eventName, handler) {
    if (!listeners.has(eventName)) listeners.set(eventName, new Set());
    listeners.get(eventName).add(handler);
    // Return the unsubscribe function
    return () => listeners.get(eventName)?.delete(handler);
  },

  publish(eventName, payload) {
    listeners.get(eventName)?.forEach(handler => handler(payload));
  },
};
```

### React binding

```jsx
// hooks/useEvent.js
import { useEffect, useRef } from "react";
import { eventBus } from "../lib/eventBus";

export function useEvent(eventName, handler) {
  // Keep latest handler in a ref so the effect doesn't resubscribe on every
  // render (inline handlers create a new reference each render)
  const handlerRef = useRef(handler);
  useEffect(() => {
    handlerRef.current = handler;
  });

  useEffect(() => {
    const unsubscribe = eventBus.subscribe(eventName, (payload) =>
      handlerRef.current(payload)
    );
    return unsubscribe; // cleanup on unmount
  }, [eventName]);
}
```

### Publisher / Subscriber components

```jsx
// Publisher
import { eventBus } from "../lib/eventBus";

function AddToCartButton({ product }) {
  return (
    <button onClick={() => eventBus.publish("cart:add", product)}>
      Add to Cart
    </button>
  );
}

// Subscriber
import { useState } from "react";
import { useEvent } from "../hooks/useEvent";

function CartBadge() {
  const [items, setItems] = useState([]);

  // Functional update — `[...items, data]` would capture a stale `items`
  useEvent("cart:add", (data) => setItems(prev => [...prev, data]));

  return <p>🛒 {items.length}</p>;
}
```

### Cross-tab broadcasting (optional)

```jsx
// lib/broadcast.js
export const crossTabChannel = new BroadcastChannel("app-events");

// In eventBus.publish, additionally:
//   crossTabChannel.postMessage({ eventName, payload });
// with a { broadcast: false } flag when re-publishing received messages,
// to avoid infinite loops.

// lib/listener.js — import once in main.jsx for its side effect
import { crossTabChannel } from "./broadcast";
import { eventBus } from "./eventBus";

crossTabChannel.onmessage = ({ data }) => {
  eventBus.publish(data.eventName, data.payload, { broadcast: false });
};
```

**When to use — and when NOT:** reach for props first, then Context. Pub-sub ONLY where those can't work: communication between separate React apps / micro-frontends, cross-tab sync, global toast/notification systems, decoupled analytics/logging. Overusing it makes data flow invisible and undebuggable.
