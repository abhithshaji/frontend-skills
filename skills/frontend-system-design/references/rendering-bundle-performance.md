# Rendering Strategies, Bundle Optimization, and Core Web Vitals

This reference covers rendering architectures (CSR, [SSR](https://nextjs.org/docs/pages/building-your-application/rendering/server-side-rendering), SSG, ISR, [RSC](https://nextjs.org/docs/app/getting-started/server-and-client-components), [PPR](https://nextjs.org/docs/15/app/getting-started/partial-prerendering)), hydration management, code splitting, lazy loading, tree shaking, resource preloading, and [Core Web Vitals](https://web.dev/articles/vitals) optimization in [React](https://react.dev/reference/react).

## 1. Rendering Strategies Comparison and Hydration

| Strategy | Where HTML is Built | When HTML is Built | Pros | Cons | Primary Use Case |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **CSR** (Client-Side Rendering) | Browser | Runtime (after JS downloads) | Cheap static hosting; instant client transitions after boot | Blank initial HTML (`<div id="root">`); poor SEO; slow initial FCP on mobile | Authenticated SaaS dashboards, internal tools, canvas apps |
| **[SSR](https://nextjs.org/docs/pages/building-your-application/rendering/server-side-rendering)** (Server-Side Rendering) | Server | Per request | Fast FCP; full SEO support; always fresh data | Higher TTFB (server compute + DB wait per request); full hydration cost | Personalized public pages, dynamic e-commerce, live feeds |
| **SSG** (Static Site Generation) | Build Server | Build / Deploy time | Fastest TTFB (served from Edge CDN); cheapest and most resilient | Requires rebuild to update content; build times scale O(N) with page count | Docs, blogs, marketing landing pages |
| **ISR** (Incremental Static Regeneration) | Build + Edge/Server | Build time + background revalidation (`revalidate: N`) | Combines CDN speed of SSG with background updates without full rebuilds | First visitor after expiry may see stale content while background regeneration runs | E-commerce catalogs with 100k+ products, news sites |
| **[RSC](https://nextjs.org/docs/app/getting-started/server-and-client-components)** (React Server Components) | Server (Serialized Flight Payload) | Build or Request time | Zero client JS bundle size for server components; direct backend DB access | Requires compatible framework (Next.js App Router / React Router v7+); boundary rules | Data-heavy shells, markdown/code renderers, complex layouts |
| **[PPR](https://nextjs.org/docs/15/app/getting-started/partial-prerendering)** (Partial Prerendering) | Static Edge Shell + Streamed Dynamic Holes | Build time (shell) + Request time (`<Suspense>` holes) | Instant static TTFB/FCP plus streamed personalized dynamic regions in one HTTP response | Requires clean `<Suspense>` boundaries around all dynamic reads (cookies/headers) | Modern e-commerce product pages (static description + live cart/inventory) |

### Preventing Hydration Mismatches in SSR/RSC

Hydration attaches React event listeners to server-rendered HTML. If the initial client render tree differs from the server HTML, React logs a hydration mismatch error and re-renders the subtree.

- **Cause**: Reading `window.innerWidth`, `localStorage`, `new Date()`, or `Math.random()` directly during render.
- **Fix**: Use `useSyncExternalStore` with a `getServerSnapshot` fallback for browser state, or defer client-only branches into `useEffect` (or pass server-seeded timestamps as props).

## 2. Code Splitting and Lazy Loading

Code splitting breaks a monolithic JavaScript bundle into smaller chunks loaded dynamically on demand, reducing initial download size, parse/compile time, and Time to Interactive (TTI).

### Pattern A: Route-Based and Component-Based Splitting

Use `React.lazy()` and `<Suspense>` for route boundaries and heavy conditional components (rich text editors, charting libraries, complex modals, below-the-fold drawers).

```tsx
import { lazy, Suspense, useState } from 'react';
import { ErrorBoundary } from 'react-error-boundary';

// Handling a Named Export cleanly with React.lazy
const HeavyAnalyticsChart = lazy(() =>
  import('./components/HeavyAnalyticsChart').then((mod) => ({
    default: mod.HeavyAnalyticsChart,
  }))
);

export function AnalyticsSection() {
  const [showChart, setShowChart] = useState(false);

  return (
    <section>
      <button
        type="button"
        onMouseEnter={() => {
          // Intent-based preload: warm the chunk cache before the user finishes clicking!
          import('./components/HeavyAnalyticsChart');
        }}
        onClick={() => setShowChart((prev) => !prev)}
      >
        {showChart ? 'Hide Analytics' : 'Show Analytics'}
      </button>

      {showChart && (
        <ErrorBoundary
          fallback={<div role="alert">Failed to load chart module. Please retry.</div>}
        >
          <Suspense fallback={<div className="h-80 animate-pulse bg-slate-200 rounded" />}>
            <HeavyAnalyticsChart />
          </Suspense>
        </ErrorBoundary>
      )}
    </section>
  );
}
```

### Pattern B: Dynamic Event-Handler Imports (Zero Render Cost)

For heavy utility libraries triggered only by user actions (e.g., PDF export, CSV parsing, zip compression), do not import them at the top of the file. Dynamically `await import()` inside the event handler:

```tsx
async function handleExportPdf(reportData: unknown) {
  const { jsPDF } = await import('jspdf');
  const doc = new jsPDF();
  doc.text(JSON.stringify(reportData), 10, 10);
  doc.save('report.pdf');
}
```

### Operational Rule: Handling Stale Deployment `ChunkLoadError`

When you deploy a new build, old hashed chunk filenames (`Dashboard.a1b2c.js`) are replaced on the CDN. Users with an existing tab open who click a lazy route will fail to fetch the deleted chunk.

- **Solution**: Listen for `vite:preloadError` (in Vite) or catch `ChunkLoadError` in a route-level Error Boundary and perform a guarded single hard navigation (`window.location.reload()` guarded by a `sessionStorage` timestamp to prevent infinite reload loops).

## 3. Tree Shaking and Bundle Hygiene

Tree shaking is dead-code elimination performed by bundlers (Vite/Rollup, Webpack, Turbopack) during production builds so unused exports are excluded from the final bundle.

### Essential Rules for Effective Tree Shaking

1. **Use ES Modules (`import` / `export`), Never CommonJS (`require`)**: Static `import { debounce } from 'es-toolkit'` allows build-time graph analysis; dynamic `require()` forces the entire module into the bundle.
2. **Eliminate Barrel Files (`index.ts`)**: A barrel file that re-exports 100 components (`export * from './Button'; export * from './DataGrid'; ...`) forces the bundler and test runner to parse the entire module graph when importing a single component. Import directly from source paths (`import { Button } from '@/components/Button'`).
3. **Declare `"sideEffects": false` in `package.json`**: Tells the bundler that importing a file and dropping unused exports has no global side effects (preserve CSS imports via `"sideEffects": ["**/*.css"]`).
4. **Avoid Monolithic Class/Object Exports**: Exporting a single giant object (`export default { fnA, fnB, fnC }`) prevents bundlers from stripping unused methods; export standalone named functions instead.

## 4. Resource Hints: Preload, Prefetch, and Preconnect

Resource hints instruct the browser to resolve connections or download critical assets before the parser discovers them naturally.

| Hint | Priority | Browser Behavior | When to Use |
| :--- | :--- | :--- | :--- |
| **[`preconnect`](https://react.dev/reference/react-dom/preconnect)** | High | Performs DNS lookup + TCP handshake + TLS negotiation ahead of time | External API domains, CDN origins, or font providers (`https://api.example.com`) |
| **[`preload`](https://react.dev/reference/react-dom/preload)** | High (Mandatory) | Downloads and caches asset immediately for the **current** page without executing | Hero LCP images, critical web fonts (`.woff2`), above-the-fold stylesheets |
| **[`preinit`](https://react.dev/reference/react-dom/preinit)** | High | Downloads **and immediately executes/applies** a script or stylesheet | Critical third-party interactive scripts or core stylesheets |
| **`prefetch`** | Low (Idle time) | Downloads resource in background idle time for **future** navigations | Next likely route JS chunk or paginated data |

### Programmatic Resource Hints in React 19 (`react-dom`)

React 19 provides native hoisted resource functions in `react-dom` ([`preconnect`](https://react.dev/reference/react-dom/preconnect), [`preload`](https://react.dev/reference/react-dom/preload), [`preinit`](https://react.dev/reference/react-dom/preinit)) that automatically deduplicate and inject `<link>` tags into `<head>`:

```tsx
import { preload, preconnect, preinit } from 'react-dom';
import { useQueryClient } from '@tanstack/react-query';

export function ProductPageShell() {
  const queryClient = useQueryClient();

  // 1. Warm DNS + TCP + TLS to API domain immediately
  preconnect('https://api.example.com');

  // 2. Preload critical LCP hero banner image
  preload('/images/hero-banner.avif', { as: 'image', fetchPriority: 'high' });

  // 3. Download and execute critical analytics/widget script
  preinit('https://cdn.example.com/critical-widget.js', { as: 'script' });

  // 4. Intent-based prefetching on link hover (both code chunk + API data!)
  const handleCheckoutHover = () => {
    import('./pages/CheckoutPage');
    queryClient.prefetchQuery({
      queryKey: ['shipping-rates'],
      queryFn: () => fetch('https://api.example.com/shipping').then((r) => r.json()),
      staleTime: 60_000,
    });
  };

  return (
    <main>
      <img
        src="/images/hero-banner.avif"
        alt="Featured Product"
        width={1200}
        height={600}
        fetchPriority="high"
        loading="eager"
      />
      <a href="/checkout" onMouseEnter={handleCheckoutHover} onFocus={handleCheckoutHover}>
        Proceed to Checkout
      </a>
    </main>
  );
}
```

## 5. Core Web Vitals (CWV) Playbook

Google evaluates real-user field performance at the **75th percentile (p75)** across three [Core Web Vitals](https://web.dev/articles/defining-core-web-vitals-thresholds):

```text
┌─────────────────────────────────────────────────────────────────────────┐
│ 1. LCP (Largest Contentful Paint) -> Loading Performance                │
│    Target: <= 2.5s | Needs Improvement: 2.5s - 4.0s | Poor: > 4.0s      │
├─────────────────────────────────────────────────────────────────────────┤
│ 2. CLS (Cumulative Layout Shift)  -> Visual Stability                   │
│    Target: <= 0.1  | Needs Improvement: 0.1 - 0.25  | Poor: > 0.25      │
├─────────────────────────────────────────────────────────────────────────┤
│ 3. INP (Interaction to Next Paint)-> Input Responsiveness               │
│    Target: <= 200ms| Needs Improvement: 200ms-500ms | Poor: > 500ms     │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1. Optimizing [LCP (Largest Contentful Paint <= 2.5s)](https://web.dev/articles/lcp)

LCP measures when the largest visible text block or image in the viewport finishes rendering. The browser continually updates the LCP candidate as larger elements paint until the user interacts with the page.

- **Never lazy-load the LCP image**: Adding `loading="lazy"` to a hero image delays discovery until after layout. Always use `loading="eager"` and `fetchPriority="high"`.
- **Eliminate Client-Side Data Waterfalls**: If the hero image URL comes from a `useEffect` fetch after JS loads, LCP will fail. Server-render the hero HTML (SSR/SSG/RSC) or [`preload`](https://react.dev/reference/react-dom/preload) the image in `<head>`.
- **Use Modern Image Formats & Responsive `srcset`**: Serve AVIF/WebP with responsive `sizes` and `srcset`.

### 2. Optimizing [CLS (Cumulative Layout Shift <= 0.1)](https://web.dev/articles/cls)

CLS measures unexpected layout shifts (`impact fraction * distance fraction`) when visible elements move without prior user interaction.

- **Always set explicit `width` and `height` (or CSS `aspect-ratio`)** on `<img>`, `<video>`, iframes, and ad slots so the browser reserves the exact bounding box before the asset loads.
- **Match Skeleton Dimensions to Final Content**: If a `<Suspense>` skeleton is `40px` tall and the hydrated card is `220px` tall, everything below shifts down by `180px`.
- **Animate only Compositor Properties**: Use `transform: translate()` and `opacity` instead of animating `top`, `height`, or `margin`.

### 3. Optimizing [INP (Interaction to Next Paint <= 200ms)](https://web.dev/articles/inp)

INP measures the latency of **all** click, tap, and keyboard interactions throughout the page's lifecycle (`Input Delay + Processing Time + Presentation Delay`).

- **Keep Urgent Event Handlers Lightweight**: Update the controlled input state immediately, and wrap expensive downstream list/chart re-renders in `startTransition` or [`useDeferredValue`](https://react.dev/reference/react/useDeferredValue):

```tsx
import { useState, useDeferredValue, memo } from 'react';

const ExpensiveResultsList = memo(function Results({ query }: { query: string }) {
  // Heavy rendering of filtered items...
  return <div>Results for {query}</div>;
});

export function ResponsiveSearch() {
  const [query, setQuery] = useState('');
  // Deferred value lags behind urgent input so typing stays < 16ms (low INP!)
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery;

  return (
    <div>
      <input
        type="search"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search catalog..."
      />
      <div style={{ opacity: isStale ? 0.7 : 1 }}>
        <ExpensiveResultsList query={deferredQuery} />
      </div>
    </div>
  );
}
```
