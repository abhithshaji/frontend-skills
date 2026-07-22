# Composition Patterns

How to structure and compose components: Compound Components, Slot Pattern, Container-Presenter, and the legacy Render Props / HOC patterns.

---

## Compound Components Pattern

**Problem it solves:** "prop soup" — a component bloated with props for every variation, rigid layout, breaks Single Responsibility (handles both structure and every content variation), poor reusability.

```jsx
// ❌ Rigid: can't do 3 buttons, no title, custom layout... without more props
function Modal({ title, body, primaryAction, secondaryAction }) { /* ... */ }
```

**Solution:** expose small "lego block" sub-components the consumer composes freely. The parent owns shared structure/state; children are dumb presentational blocks.

```jsx
function Modal({ children, isOpen, onClose }) {
  if (!isOpen) return null;
  return (
    <div className="modal-backdrop">
      <div className="modal-container">
        {children}
        <button className="modal-close" onClick={onClose}>✖</button>
      </div>
    </div>
  );
}

Modal.Header = function ModalHeader({ children }) {
  return <div className="modal-header">{children}</div>;
};
Modal.Body = function ModalBody({ children }) {
  return <div className="modal-body">{children}</div>;
};
Modal.Footer = function ModalFooter({ children }) {
  return <div className="modal-footer">{children}</div>;
};

export default Modal;

// Usage — any structure the consumer wants:
<Modal isOpen={open} onClose={close}>
  <Modal.Header>Delete account?</Modal.Header>
  <Modal.Body>This cannot be undone.</Modal.Body>
  <Modal.Footer>
    <button>Cancel</button>
    <button>Maybe later</button>
    <button>Delete</button>
  </Modal.Footer>
</Modal>
```

### Sharing implicit state between parent and children

When sub-components need the parent's state (active tab, open accordion item, form values), share it via context — NOT by cloning children or prop drilling:

```jsx
const AccordionContext = createContext(null);

function Accordion({ children, allowMultiple = false }) {
  const [openItems, setOpenItems] = useState(new Set());

  const toggle = (id) =>
    setOpenItems(prev => {
      const next = new Set(allowMultiple ? prev : []);
      prev.has(id) ? next.delete(id) : next.add(id);
      return next;
    });

  return (
    <AccordionContext.Provider value={{ openItems, toggle }}>
      <div className="accordion">{children}</div>
    </AccordionContext.Provider>
  );
}

function AccordionItem({ id, title, children }) {
  const ctx = useContext(AccordionContext);
  if (!ctx) throw new Error("Accordion.Item must be used within <Accordion>");
  const isOpen = ctx.openItems.has(id);

  return (
    <div className="accordion-item">
      <button aria-expanded={isOpen} onClick={() => ctx.toggle(id)}>
        {title}
      </button>
      {isOpen && <div className="accordion-content">{children}</div>}
    </div>
  );
}

Accordion.Item = AccordionItem;
```

**Best practices:**
- Throw a descriptive error when a sub-component is used outside its parent (see guard above).
- `Object.assign(Root, { Item, Header })` in an index file also works and plays better with tree-shaking and React.forwardRef than mutating the function object.
- Don't overuse: deeply nested compound structures get hard to debug. Simple components with 2–3 props don't need this.

**Use when:** modals, tabs, accordions, menus, dropdowns, form field groups — any "family" of components that work together.

---

## Slot Pattern

A **slot** is a placeholder region inside a component where the consumer places arbitrary UI. Unlike compound components, slots don't share state — they're purely about layout placement.

### 1. Single unnamed slot — `children`

```jsx
function Card({ children }) {
  return <div className="card">{children}</div>;
}
```

### 2. Named slots — React nodes as props

```jsx
function CardWithSlots({ header, content, footer }) {
  return (
    <div className="card">
      <div className="card-header">{header}</div>
      <div className="card-content">{content}</div>
      <div className="card-footer">{footer}</div>
    </div>
  );
}

// Usage
<CardWithSlots
  header={<h3>User Profile</h3>}
  content={<p>Biography and details...</p>}
  footer={<button>Follow</button>}
/>
```

Pros: simple, explicit. Cons: awkward when the number of slots grows.

### 3. Slot map — one object prop of named regions

```jsx
function SlotsLayout({ slots = {} }) {
  return (
    <div className="layout">
      <aside>{slots.sidebar}</aside>
      <main>
        <header>{slots.header}</header>
        <section>{slots.content}</section>
        <footer>{slots.footer}</footer>
      </main>
    </div>
  );
}

<SlotsLayout
  slots={{
    header: <h1>Title</h1>,
    sidebar: <nav>Menu</nav>,
    content: <Article />,
    footer: <small>Foot</small>,
  }}
/>
```

**Use when:** page layouts, cards, panels, list items with configurable leading/trailing content (e.g. `startIcon` / `endIcon` props are slots).

**Slot vs Compound:** slots for placement only; compound components when regions must coordinate through shared state.

---

## Container-Presenter Pattern

**Problem:** one component does the API call, manages state, handles form logic, AND renders loading/error/data UI. That violates SRP, kills reusability (the inline loading/error UI can't be reused), and makes testing painful (mock axios + every state + the UI in one test).

**Solution:** split into two layers:
- **Container** — data fetching, state, event handlers. Renders NO markup of its own; passes data down.
- **Presenter(s)** — pure props-in, JSX-out. No fetching, no business logic. Can be split into multiple smaller presenters.

```jsx
// Container — all the logic, no UI
function UserListContainer() {
  const { data: users, loading, error } = useFetch("/api/users");

  if (loading) return <Spinner />;
  if (error) return <ErrorMessage message={error} />;
  return <UserList users={users} />;
}

// Presenter — dumb, reusable, trivially testable
function UserList({ users }) {
  return (
    <ul>
      {users.map(u => <li key={u.id}>{u.name}</li>)}
    </ul>
  );
}
```

**Modern form:** the "container" is usually just a custom hook (`useUsers()`) consumed by a thin component — same separation, less nesting. With Suspense, loading/error states move up to `<Suspense>`/`<ErrorBoundary>` wrappers (see [async-patterns.md](async-patterns.md)).

**Use when:** data-heavy dashboards, analytics views, form-heavy apps — anywhere testability and reuse of presentational pieces matters.

---

## Render Props Pattern (legacy — prefer hooks)

A prop whose value is a **function that returns JSX**. The component owns the logic and calls your function with data; you decide the rendering. Solved logic-sharing before hooks existed.

```jsx
function MouseTracker({ render }) {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  return (
    <div onMouseMove={(e) => setPos({ x: e.clientX, y: e.clientY })}>
      {render(pos)}
    </div>
  );
}

// Usage — consumer controls rendering:
<MouseTracker render={({ x, y }) => <p>Mouse at {x}, {y}</p>} />

// Variant: children as a function
<MouseTracker>{({ x, y }) => <Cat x={x} y={y} />}</MouseTracker>
```

**Modern equivalent:** a custom hook (`const { x, y } = useMousePosition()`). Only write render props to match an existing codebase, or when a component must control *where* in the tree the injected UI renders (e.g. virtualization row renderers — `react-window`'s children function is a render prop).

---

## Higher-Order Components (legacy — prefer hooks)

An **HOC** is a function that takes a component and returns a new component with extra behavior injected (data fetching, auth, logging). Naming convention: `withX` — `withDataFetching`, `withAuth`.

```jsx
export function withDataFetching(WrappedComponent, url) {
  return function WithDataFetching(props) {
    const [data, setData] = useState([]);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);

    useEffect(() => {
      const controller = new AbortController();
      (async () => {
        try {
          const res = await fetch(url, { signal: controller.signal });
          if (!res.ok) throw new Error("Failed to fetch data");
          setData(await res.json());
        } catch (err) {
          if (err.name !== "AbortError") setError(err.message);
        } finally {
          setLoading(false);
        }
      })();
      return () => controller.abort();
    }, []);

    if (loading) return <p>Loading…</p>;
    if (error) return <p>Error: {error}</p>;
    return <WrappedComponent data={data} {...props} />;
  };
}

// One HOC, many enhanced components:
const MovieListWithData = withDataFetching(MovieList, "/api/movies");
const MovieAnalyticsWithData = withDataFetching(MovieAnalytics, "/api/movies");
```

**Pitfalls:** wrapper hell in DevTools, prop-name collisions, lost refs (needs `forwardRef`), static methods not copied. These are why hooks replaced HOCs. Recognize the pattern in legacy code; write `useFetch()` + composition in new code.
