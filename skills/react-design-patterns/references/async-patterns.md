# Async & Architecture Patterns

Suspense + `use()`, Error Boundary, Optimistic UI, Facade Hook, and Strategy + Hook Factory.

---

## Suspense Pattern

**Problem — fetch-on-render waterfalls.** With `useEffect` fetching, each component renders, THEN fetches, then dependent fetches chain off state, producing sequential waterfalls and one giant "Loading dashboard..." gate:

```jsx
// ❌ Waterfall: user → then orders → then analytics, all gating the whole page
useEffect(() => { fetchUser().then(setUser); }, []);
useEffect(() => { if (user) fetchOrders(user.id).then(setOrders); }, [user]);
useEffect(() => { if (user) fetchAnalytics(user.id).then(setAnalytics); }, [user]);
if (!user || !orders || !analytics) return <p>Loading dashboard...</p>;
```

**Solution — render-as-you-fetch.** Start fetches *before/outside* render, then let each component *suspend* until its data is ready. `<Suspense>` declaratively shows a fallback while children suspend. `use()` (React 19) reads a promise inside a component and suspends until it resolves.

### 1. Create resources (start fetching early)

```jsx
// resources/userResource.js
import { fetchAnalytics, fetchOrders, fetchUser } from "../api";

let userPromise, ordersPromise, analyticsPromise;

export function createUserResources() {
  userPromise = fetchUser();
  // Dependent fetches chain off the promise, not off component renders
  ordersPromise = userPromise.then(user => fetchOrders(user.id));
  analyticsPromise = userPromise.then(user => fetchAnalytics(user.id));
}

export function getUserResources() {
  return { userPromise, ordersPromise, analyticsPromise };
}
```

```jsx
// main.jsx — kick off before rendering
import { createUserResources } from "./resources/userResource";
createUserResources();
```

> **Important:** promises passed to `use()` must be stable/cached (module-level here). Creating a promise *inside* render (`use(fetchUser())`) causes an infinite suspend-refetch loop. In production apps, use a Suspense-enabled library (TanStack Query's `useSuspenseQuery`, or a framework loader like React Router / Next.js) instead of hand-rolled caches.

### 2. Components read with `use()`

```jsx
import { use } from "react";
import { getUserResources } from "../resources/userResource";

export default function Profile() {
  const { userPromise } = getUserResources();
  const user = use(userPromise); // suspends until resolved, throws on reject
  return <h2>Welcome, {user.name}</h2>;
}
```

### 3. Independent Suspense + ErrorBoundary per widget

```jsx
export default function Dashboard() {
  return (
    <div>
      <h1>📊 Dashboard</h1>

      <ErrorBoundary fallback={ProfileError}>
        <Suspense fallback={<ProfileSkeleton />}>
          <Profile />
        </Suspense>
      </ErrorBoundary>

      <ErrorBoundary fallback={OrdersError}>
        <Suspense fallback={<OrdersSkeleton />}>
          <Orders />
        </Suspense>
      </ErrorBoundary>

      <ErrorBoundary fallback={AnalyticsError}>
        <Suspense fallback={<AnalyticsSkeleton />}>
          <Analytics />
        </Suspense>
      </ErrorBoundary>
    </div>
  );
}
```

Each widget streams in independently — no all-or-nothing gate. Use skeletons matching the final layout to avoid layout shift.

---

## Error Boundary Pattern

React's default: a rendering error **anywhere** unmounts the **entire** tree. An error boundary catches render errors in its child subtree and swaps that subtree for a fallback UI.

Must be a **class component** — `getDerivedStateFromError`/`componentDidCatch` have no hook equivalents (yet).

```jsx
import React from "react";

class ErrorBoundary extends React.Component {
  state = { hasError: false, error: null, retryKey: 0 };

  // Called during render when a child throws — return new state
  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  // Called after — side effects OK: log to a reporting service here
  componentDidCatch(error, info) {
    console.error("Error:", error.message);
    console.error("Component stack:", info.componentStack);
  }

  handleRetry = () => {
    // Bumping retryKey remounts the children subtree fresh
    this.setState(prev => ({
      hasError: false,
      error: null,
      retryKey: prev.retryKey + 1,
    }));
  };

  render() {
    const { hasError, error, retryKey } = this.state;
    const { fallback: Fallback, children } = this.props;

    if (hasError) return <Fallback error={error} onRetry={this.handleRetry} />;
    return <React.Fragment key={retryKey}>{children}</React.Fragment>;
  }
}

export default ErrorBoundary;
```

```jsx
// Fallback UI — a plain component receiving error + retry
const ErrorFallback = ({ error, onRetry }) => (
  <div role="alert" className="border border-red-600 p-2 rounded">
    <h3>{error?.message}</h3>
    <button onClick={onRetry}>Retry</button>
  </div>
);

// Usage
<ErrorBoundary fallback={ErrorFallback}>
  <StatsWidget />
</ErrorBoundary>
```

**Rules:**
- **Never wrap only the app root.** If one widget fails, the whole app becomes fallback UI. Place boundaries around independent widgets/routes; a root-level one is only the last resort.
- Error boundaries do NOT catch: event handler errors (use try/catch), async errors outside render, SSR errors. But a promise rejection surfaced through `use()` IS caught (see Suspense above — retry should also recreate the failed resource).
- In production, prefer the `react-error-boundary` package: function-component API, `onReset`, `resetKeys`, `useErrorBoundary()` for manually surfacing async errors.

---

## Optimistic UI Pattern

Update the UI **immediately**, then run the server call; reconcile (commit or revert) when it completes. Feels instant for high-frequency, likely-to-succeed actions.

**`useOptimistic` (React 19):**

```jsx
const [optimisticState, addOptimistic] = useOptimistic(currentState, updateFn);
```

- `currentState` — the real committed state (from `useState`, props, or server).
- `updateFn(currentState, input)` — pure; returns the next optimistic state.
- `addOptimistic(input)` — apply an optimistic update. **Must be called inside a transition or form action.**
- When the transition/action finishes, `optimisticState` automatically snaps back to whatever `currentState` is — success means you committed the real state (no visible change); failure means it reverts on its own. **You never manually roll back.**

```jsx
import { useOptimistic, useState, startTransition } from "react";

async function sendLikeToServer(postId) {
  await new Promise(r => setTimeout(r, 700));
  if (Math.random() < 0.2) throw new Error("Network failed");
  return { success: true };
}

export default function LikeButton({ postId, initialLikes = 0 }) {
  const [likes, setLikes] = useState(initialLikes); // real committed state
  const [error, setError] = useState(null);
  const [optimisticLikes, addOptimisticLike] = useOptimistic(
    likes,
    (current, delta) => current + delta
  );

  const handleLike = () => {
    setError(null);
    startTransition(async () => {
      addOptimisticLike(1); // instant UI bump
      try {
        await sendLikeToServer(postId);
        setLikes(prev => prev + 1); // commit — optimistic now matches real
      } catch (err) {
        // No manual rollback needed: when the transition ends, optimisticLikes
        // reverts to `likes`. Just tell the user.
        setError("Couldn't like — try again");
      }
    });
  };

  return (
    <div>
      <button onClick={handleLike}>❤️ {optimisticLikes}</button>
      {error && <span role="alert">{error}</span>}
    </div>
  );
}
```

With `<form action={...}>` (React 19 form actions) the action IS the transition — call `addOptimistic` at the top of the action function.

**Pre-React-19 / library approach:** TanStack Query `onMutate` (apply) + `onError` (rollback with snapshot) + `onSettled` (invalidate).

**Use for:** likes, claps, votes, add-to-cart, toggles, collaborative editing. **Avoid for:** payments, deletions, anything where showing premature success is harmful.

---

## Facade Pattern

Provide a **simple interface over a complex subsystem** — like a hotel reception desk: you ask reception for food/taxi/room service instead of calling each service yourself.

In React: a **facade is a custom hook that composes several hooks** and exposes exactly what the consuming component needs — fewer imports, fewer states to juggle, and the component no longer knows *which* underlying systems exist.

```jsx
// hooks/facade/useDashboardFacade.js
import { useAuth } from "../useAuth";
import { useFeatureFlags } from "../useFeatureFlags";
import { usePermissions } from "../usePermissions";
import { useUser } from "../useUser";

export function useDashboardFacade() {
  const { isAuthenticated, logout } = useAuth();
  const { user, isLoading: userLoading } = useUser();
  const { permissions, isLoading: permLoading } = usePermissions();
  const { flags } = useFeatureFlags();

  return {
    isAuthenticated,
    isLoading: userLoading || permLoading,
    user,
    canEditDashboard: permissions?.includes("EDIT_DASHBOARD") ?? false,
    showBetaDashboard: flags?.enableBetaDashboard ?? false,
    logout,
  };
}
```

```jsx
// Dashboard.jsx — one import, one call, zero knowledge of the subsystems
export default function Dashboard() {
  const { isAuthenticated, isLoading, user, canEditDashboard, showBetaDashboard, logout } =
    useDashboardFacade();

  if (isLoading) return <p>Loading...</p>;
  if (!isAuthenticated) return <p>Please login</p>;

  return (
    <div>
      <h1>Welcome {user.name}</h1>
      {canEditDashboard && <button>Edit Dashboard</button>}
      {showBetaDashboard && <p>Beta Feature Enabled</p>}
      <button onClick={logout}>Logout</button>
    </div>
  );
}
```

**Benefits:** swap implementations (new auth provider) behind the facade without touching consumers; derived flags (`canEditDashboard`) computed once, consistently. **Caveat:** every consumer subscribes to ALL composed hooks' updates — don't make one mega-facade for the whole app; scope facades per feature/screen.

---

## Strategy Pattern & Hook Factory

**Problem:** interchangeable behaviors (OTP vs SSO vs password auth; Stripe vs Razorpay payment) implemented with `if/else` chains scattered through components — every new variant means editing every component.

**Strategy pattern:** ① define a common interface each strategy implements → ② implement each strategy separately → ③ select the strategy by key at runtime → ④ consumers call the interface, unaware of which strategy runs.

In React each strategy is a **custom hook** with the same return shape, and a **hook factory** maps keys to hooks:

```jsx
// hooks/useOtpAuth.js — every auth hook returns the SAME shape
export function useOtpAuth() {
  return {
    login() { /* OTP login logic */ },
    logout() { /* ... */ },
  };
}

// hooks/useSsoAuth.js
export function useSsoAuth() {
  return {
    login() { /* SSO redirect logic */ },
    logout() { /* ... */ },
  };
}
```

```jsx
// hookFactory.js
import { useOtpAuth } from "./hooks/useOtpAuth";
import { useSsoAuth } from "./hooks/useSsoAuth";

const authHookMap = { otp: useOtpAuth, sso: useSsoAuth };

export function createAuthHook(type) {
  const hook = authHookMap[type];
  if (!hook) throw new Error(`Unknown auth strategy: ${type}`);
  return hook;
}
```

```jsx
// Login.jsx
import { createAuthHook } from "./hookFactory";

function Login({ loginType }) {
  const useAuth = createAuthHook(loginType); // select BEFORE calling
  const auth = useAuth();                    // then call normally
  return <button onClick={auth.login}>Login</button>;
}
```

> ⚠️ **Rules-of-Hooks caveat:** if `loginType` changes between renders, a *different* hook runs in the same component — that breaks hook ordering. Either (a) choose the strategy once at startup, or (b) force a remount when it changes: `<Login key={loginType} loginType={loginType} />`.

**Safer app-wide setup — choose once via Provider:**

```jsx
// AuthProvider.jsx — strategy fixed at app startup
import { AuthContext } from "./context";
import { createAuthHook } from "./hookFactory";

export function AuthProvider({ strategy, children }) {
  const useAuth = createAuthHook(strategy);
  const auth = useAuth(); // call it HERE, share the result
  return <AuthContext.Provider value={auth}>{children}</AuthContext.Provider>;
}

// App.jsx
<AuthProvider strategy="otp">
  <LoginButton />
  <Dashboard />
</AuthProvider>
```

**Use when:** multiple interchangeable implementations of one capability — auth methods, payment providers, storage backends, export formats, pricing rules. Adding a variant = adding one hook + one map entry; zero component edits.
