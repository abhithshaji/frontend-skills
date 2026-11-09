# Accessibility (WCAG 2.2 AA), Security, Resilience, Observability, and i18n

This reference covers [WCAG 2.2](https://www.w3.org/TR/WCAG22/) accessibility implementation in [React](https://react.dev/reference/react), keyboard and screen-reader patterns, granular Error Boundaries, network resilience, frontend security (XSS, CSRF, CSP, Auth), observability, and internationalization.

## 1. Accessibility (a11y) & [WCAG 2.2](https://www.w3.org/TR/WCAG22/) Architecture

### Core Mental Models

- **The Golden Rule of ARIA**: *No ARIA is better than bad ARIA.* Always use native semantic HTML elements (`<button>`, `<a>`, `<input>`, `<dialog>`, `<main>`, `<nav>`) before reaching for custom `div` elements with `role` attributes. Native elements provide built-in keyboard handling and accessibility tree semantics for free.
- **Tab Order and `tabIndex` Rules**:
  - `tabIndex={0}`: Inserts a custom interactive element into the natural DOM tab order.
  - `tabIndex={-1}`: Removes the element from the `Tab` key sequence, but allows it to receive **programmatic focus** via `ref.current?.focus()` (essential for SPA route headings, modals, and roving tabindex menus).
  - **NEVER use `tabIndex >= 1`** (e.g., `tabIndex={1}` or `tabIndex={2}`): Positive `tabIndex` values hijack the page's natural focus sequence, forcing keyboard users through those elements before anything else on the page. Always order elements naturally in the DOM instead.
- **Custom Button Keyboard Semantics**: If you must make a non-button element interactive, add `role="button"`, `tabIndex={0}`, and handle **both** `Enter` and `Space` (`" "`), calling `e.preventDefault()` on `Space` so the page does not scroll down.

### Semantic Landmark Blueprint & SPA Route Announcements

Screen reader users navigate via landmarks (`<header>`, `<nav aria-label="...">`, `<main id="main-content">`, `<aside>`, `<section aria-labelledby="...">`, `<footer>`) and heading hierarchies (`<h1>` -> `<h2>` -> `<h3>`; never skip heading levels for visual sizing).

In a Client-Side Rendered (CSR) SPA, route changes do not trigger a browser page reload, leaving screen readers silent. On route change:

1. Update `document.title` immediately.
2. Either shift focus programmatically (`h1Ref.current?.focus()`) to the page's main `<h1 tabIndex={-1} className="outline-none">` or announce the new page title inside an `aria-live="polite"` region.

### Visual vs Assistive Hiding Matrix

| Technique | CSS / Attribute | Visually Seen? | Read by Screen Reader? | Focusable via Tab? |
| :--- | :--- | :--- | :--- | :--- |
| **Complete Hide** | `display: none`, `visibility: hidden`, or `hidden` | No | No | No |
| **Decorative Only** | `aria-hidden="true"` (e.g., decorative SVG icons) | Yes | **No** | No (Never put on focusable elements!) |
| **Screen Reader Only** | Tailwind `sr-only` class | **No** | **Yes** | Only if interactive (e.g., Skip-to-Content link) |

```css
/* Standard .sr-only implementation when Tailwind is unavailable */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

### Accessible Form Field with Dynamic Validation (`useId`)

Connect `<label>`, helper text, and live error messages deterministically using React's `useId()` ([React Hooks](https://react.dev/reference/react/hooks)), `aria-invalid`, and `aria-describedby`:

```tsx
import { useId, useState } from 'react';

interface AccessibleInputProps {
  label: string;
  helperText?: string;
  validate?: (val: string) => string | null;
}

export function AccessibleInput({ label, helperText, validate }: AccessibleInputProps) {
  const [value, setValue] = useState('');
  const [error, setError] = useState<string | null>(null);

  const inputId = useId();
  const helperId = useId();
  const errorId = useId();

  const describedBy = [error ? errorId : null, helperText ? helperId : null]
    .filter(Boolean)
    .join(' ') || undefined;

  return (
    <div className="flex flex-col gap-1">
      <label htmlFor={inputId} className="font-medium">
        {label}
      </label>
      <input
        id={inputId}
        type="text"
        value={value}
        onChange={(e) => setValue(e.target.value)}
        onBlur={() => validate && setError(validate(value))}
        aria-invalid={Boolean(error)}
        aria-describedby={describedBy}
      />
      {helperText && !error && (
        <span id={helperId} className="text-sm text-slate-600">{helperText}</span>
      )}
      {error && (
        <span id={errorId} role="alert" className="text-sm text-red-600">{error}</span>
      )}
    </div>
  );
}
```

### Advanced Interaction: Menu Arrow-Key Navigation & Focus Restoration

Composite widgets (dropdowns, tabs, comboboxes, modals) must trap or manage internal focus with arrow keys (`ArrowDown`, `ArrowUp`, `Home`, `End`), close on `Escape`, and restore focus back to the trigger button when closed:

```tsx
import { useState, useRef, useEffect, KeyboardEvent } from 'react';

export function AccessibleMenu({ items }: { items: Array<{ label: string; onSelect: () => void }> }) {
  const [isOpen, setIsOpen] = useState(false);
  const triggerRef = useRef<HTMLButtonElement>(null);
  const menuRef = useRef<HTMLUListElement>(null);

  const closeMenu = () => {
    setIsOpen(false);
    triggerRef.current?.focus(); // Always restore focus to trigger!
  };

  useEffect(() => {
    if (isOpen) {
      const first = menuRef.current?.querySelector<HTMLElement>('[role="menuitem"]');
      first?.focus();
    }
  }, [isOpen]);

  const handleKeyDown = (e: KeyboardEvent<HTMLDivElement>) => {
    if (!isOpen) return;
    const nodes = Array.from(
      menuRef.current?.querySelectorAll<HTMLElement>('[role="menuitem"]') ?? []
    );
    const idx = nodes.indexOf(document.activeElement as HTMLElement);

    switch (e.key) {
      case 'Escape':
        e.preventDefault();
        closeMenu();
        break;
      case 'ArrowDown':
        e.preventDefault();
        nodes[(idx + 1) % nodes.length]?.focus();
        break;
      case 'ArrowUp':
        e.preventDefault();
        nodes[(idx - 1 + nodes.length) % nodes.length]?.focus();
        break;
      case 'Home':
        e.preventDefault();
        nodes[0]?.focus();
        break;
      case 'End':
        e.preventDefault();
        nodes[nodes.length - 1]?.focus();
        break;
    }
  };

  return (
    <div onKeyDown={handleKeyDown} className="relative inline-block">
      <button
        ref={triggerRef}
        type="button"
        aria-haspopup="menu"
        aria-expanded={isOpen}
        onClick={() => setIsOpen((o) => !o)}
      >
        Actions
      </button>
      {isOpen && (
        <ul ref={menuRef} role="menu" className="absolute mt-1 border rounded bg-white shadow">
          {items.map((item) => (
            <li
              key={item.label}
              role="menuitem"
              tabIndex={-1}
              onClick={() => {
                item.onSelect();
                closeMenu();
              }}
              onKeyDown={(e) => {
                if (e.key === 'Enter' || e.key === ' ') {
                  e.preventDefault();
                  item.onSelect();
                  closeMenu();
                }
              }}
              className="px-4 py-2 cursor-pointer focus:bg-teal-600 focus:text-white"
            >
              {item.label}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

### [What's New in WCAG 2.2](https://www.w3.org/WAI/standards-guidelines/wcag/new-in-22/) Checklist

- **[Target Size (Minimum) (SC 2.5.8 - Level AA)](https://www.w3.org/WAI/WCAG22/Understanding/target-size-minimum.html)**: Interactive touch/click targets must be at least **24x24 CSS pixels** (or have at least 24px spacing clearance).
- **[Focus Not Obscured (Minimum) (SC 2.4.11 - Level AA)](https://www.w3.org/WAI/WCAG22/Understanding/focus-not-obscured-minimum.html)**: When a UI component receives keyboard focus, it must not be entirely hidden by sticky headers, footers, or non-modal banners.
- **[Focus Appearance (SC 2.4.13 - Level AAA)](https://www.w3.org/WAI/WCAG22/Understanding/focus-appearance.html) & Focus Visible (Level AA)**: Provide clear `:focus-visible` indicators with at least a **3:1 contrast ratio** and sufficient perimeter area.
- **Dragging Movements (SC 2.5.7 - Level AA)**: Any drag-and-drop operation (such as Kanban card reordering) must also be achievable with a single pointer or keyboard buttons without dragging.
- **Redundant Entry (SC 3.3.7 - Level A) & Accessible Authentication (SC 3.3.8 - Level AA)**: Do not force users to re-type data entered earlier in the same flow, and never block copy/paste or password managers on authentication inputs.

## 2. Resilience, Error Boundaries, and Network Fault Tolerance

### Granular Error Boundary Hierarchy

Never rely on a single root Error Boundary that turns a broken sidebar widget into a full-page crash screen. Place Error Boundaries at three distinct levels:

1. **Widget / Card Boundary**: Isolates non-critical regions (analytics charts, recommendation rails, comments) with an inline "Retry" button bound to `resetKeys`.
2. **Route / Page Boundary**: Catches page-level rendering or `ChunkLoadError` failures while keeping navigation chrome alive.
3. **App Root Boundary**: Last-resort catastrophic fallback with error telemetry reporting.

### Network Fault Tolerance Rules

- **Request Cancellation (`AbortController`)**: Always forward `signal` provided by TanStack Query `queryFn: ({ signal }) => fetch(url, { signal })` so unmounted or superseded requests abort immediately.
- **Retry Discipline**: Only retry transient network errors and `502`/`503`/`504`/`429` responses using exponential backoff with jitter. Never auto-retry `400`, `401`, `403`, or `422` validation errors.

## 3. Frontend Security Architecture (XSS, CSRF, CSP, Auth)

1. **Cross-Site Scripting (XSS) Prevention**:
   - React escapes string variables in JSX (`{userInput}`) automatically, **except** inside `dangerouslySetInnerHTML`, `<a href={untrustedUrl}>` (which can execute `javascript:alert(1)`), and unescaped SSR `<script>` state payloads.
   - Always sanitize HTML with **[DOMPurify](https://github.com/cure53/dompurify)** (`DOMPurify.sanitize(html)`) before passing to `dangerouslySetInnerHTML`, and validate that dynamic URLs start with `https://`, `http://`, or `/`.
2. **Authentication & Token Storage**:
   - Avoid storing long-lived Refresh Tokens or sensitive JWTs in [`localStorage`](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage) (accessible to any XSS payload or compromised npm script).
   - Store Refresh Tokens in **`HttpOnly; Secure; SameSite=Lax/Strict`** cookies, keep short-lived Access Tokens in client memory (or rely on BFF cookie sessions), and deduplicate concurrent `401 Unauthorized` token-refresh calls using a shared Promise mutex.
3. **Content Security Policy (CSP)**: Enforce strict `Content-Security-Policy` headers (`script-src 'self' 'nonce-...'`, `frame-ancestors 'none'`) to block unauthorized script execution and clickjacking.

## 4. Production Observability & Internationalization (i18n)

- **Real User Monitoring (RUM)**: Capture field [Core Web Vitals](https://web.dev/articles/vitals) (`onLCP`, `onINP`, `onCLS` from the `web-vitals` package) and flush metrics via `navigator.sendBeacon()` on page hide. Propagate an `X-Request-ID` or `traceparent` header from React API calls to backend logs for end-to-end debugging.
- **Internationalization (i18n) & RTL Layout**:
  - Dynamically code-split locale dictionaries (`await import(\`./locales/\${locale}.json\`)`) so users download only their active language.
  - Format dates, currencies, and plurals with native browser **`Intl`** APIs (`Intl.NumberFormat`, `Intl.DateTimeFormat`, `Intl.RelativeTimeFormat`) instead of heavy legacy date bundles.
  - Use **CSS Logical Properties** (`margin-inline-start`, `padding-inline`, `text-align: start`, `inset-inline-end`, or Tailwind `ms-*`, `me-*`, `ps-*`, `pe-*`) instead of physical `left`/`right` so layouts flip automatically for RTL languages (`dir="rtl"`).
