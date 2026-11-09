# Large Lists, Virtualization, Pagination, Reordering, and Web Workers

This reference covers [DOM virtualization](https://tanstack.com/virtual/latest/docs/introduction), pagination strategies (Offset vs Cursor vs Infinite Scroll), O(1) drag-and-drop list reordering at scale (LexoRank / Fractional Indexing), and offloading CPU-intensive work in data-heavy [React](https://react.dev/reference/react) apps to [Web Workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API).

## 1. DOM Virtualization (Windowing)

When rendering lists with hundreds or thousands of complex items, creating 10,000 DOM nodes causes massive layout recalculation overhead, high memory usage, and janky scrolling. **Virtualization** renders only the items currently visible inside the scroll container viewport plus a small `overscan` buffer (3 to 5 items above and below to prevent blank flashing during fast flicks).

### Pattern A: Headless Virtualization with [`@tanstack/react-virtual`](https://tanstack.com/virtual/latest/docs/introduction) (Fixed & Dynamic Heights)

[`@tanstack/react-virtual`](https://tanstack.com/virtual/latest/docs/api/virtualizer) is lightweight, headless, and supports dynamic row measurement via `measureElement` (powered by `ResizeObserver`) as well as `anchorTo: 'end'` for chat/AI streams and log feeds:

```tsx
import { useRef } from 'react';
import { useVirtualizer } from '@tanstack/react-virtual';

export function VirtualizedLogFeed({ items }: { items: Array<{ id: string; text: string }> }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const rowVirtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    getItemKey: (index) => items[index].id,
    estimateSize: () => 64, // Estimated row height in px
    overscan: 5,            // Render 5 extra rows above/below viewport
  });

  return (
    <div
      ref={parentRef}
      style={{ height: '600px', overflow: 'auto', position: 'relative' }}
    >
      {/* Total virtual height container keeps native scrollbar accurate */}
      <div
        style={{
          height: `${rowVirtualizer.getTotalSize()}px`,
          width: '100%',
          position: 'relative',
        }}
      >
        {rowVirtualizer.getVirtualItems().map((virtualRow) => {
          const item = items[virtualRow.index];
          return (
            <div
              key={virtualRow.key}
              data-index={virtualRow.index}
              ref={rowVirtualizer.measureElement} // Dynamically measures actual DOM height!
              style={{
                position: 'absolute',
                top: 0,
                left: 0,
                width: '100%',
                transform: `translateY(${virtualRow.start}px)`,
              }}
            >
              {item.text}
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

### Pattern B: `react-window` FixedSizeList & CSS `content-visibility: auto`

- For simple fixed-height rows, `react-window` (`FixedSizeList`) provides a declarative wrapper (`itemCount`, `itemSize`, `overscanCount={5}`, `itemData`).
- For moderate lists (100 to 500 items) where preserving native browser `Ctrl+F` find-in-page and SEO is required, apply CSS `content-visibility: auto; contain-intrinsic-size: 0 80px;` to row items so the browser skips layout and paint for off-screen elements natively.

## 2. Pagination Architecture: Offset vs Cursor vs Infinite Scroll

| Feature | Offset Pagination | Cursor (Keyset) Pagination | Infinite Scroll |
| :--- | :--- | :--- | :--- |
| **UX Pattern** | Numbered page links (`1`, `2`, `...`, `40`) | Sequential "Next / Previous" | Continuous fetch on scroll proximity |
| **SQL / Backend Strategy** | `LIMIT 20 OFFSET 1000` | `WHERE (created_at, id) < (:cursor) LIMIT 20` | Frontend UX backed by **Cursor API** |
| **Deep Page Latency** | Degrades linearly **O(N)** (DB scans and discards skipped rows) | Constant **O(1)** (Direct B-Tree index seek) | Constant **O(1)** when backed by Cursors |
| **Real-Time Data Drift** | **High risk**: New row at top shifts window, duplicating or skipping items | **Immune**: Anchored to a specific record ID/timestamp | **Immune** when backed by Cursors |
| **Random Page Jumps** | Yes (Can jump directly to Page 25) | No (Strictly sequential traversal) | No (Sequential discovery) |

### Architectural Law: Never Back Infinite Scroll with Offset Pagination

If a user scrolls deep into an offset-backed feed while new items are inserted at the top of the database, every subsequent page shifts down, causing duplicate items (triggering React duplicate `key` bugs and layout jumps) and progressively slower database queries. Always pair Infinite Scroll with **Cursor Pagination** and an `IntersectionObserver`:

```tsx
import { useEffect, useRef } from 'react';
import { useInfiniteQuery } from '@tanstack/react-query';

interface PageResponse {
  items: Array<{ id: string; title: string }>;
  nextCursor: string | null;
}

export function CursorInfiniteFeed() {
  const sentinelRef = useRef<HTMLDivElement>(null);

  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useInfiniteQuery({
    queryKey: ['feed'],
    queryFn: async ({ pageParam }): Promise<PageResponse> => {
      const url = pageParam ? `/api/feed?cursor=${encodeURIComponent(pageParam)}` : '/api/feed';
      const res = await fetch(url);
      if (!res.ok) throw new Error('Failed to load feed');
      return res.json();
    },
    initialPageParam: null as string | null,
    getNextPageParam: (lastPage) => lastPage.nextCursor,
    maxPages: 10, // Optional: bounds client memory by dropping oldest pages on deep scroll
  });

  useEffect(() => {
    const el = sentinelRef.current;
    if (!el) return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting && hasNextPage && !isFetchingNextPage) {
          fetchNextPage();
        }
      },
      { rootMargin: '300px' } // Prefetch 300px before user hits bottom
    );

    observer.observe(el);
    return () => observer.disconnect();
  }, [hasNextPage, isFetchingNextPage, fetchNextPage]);

  return (
    <div>
      {data?.pages.flatMap((p) => p.items).map((item) => (
        <article key={item.id} className="p-4 border-b">{item.title}</article>
      ))}
      <div ref={sentinelRef} className="h-10 flex items-center justify-center">
        {isFetchingNextPage && <span>Loading more...</span>}
      </div>
    </div>
  );
}
```

## 3. Reordering Lists at Scale (Kanban Boards & Ranked Queues)

In a Kanban board or playlist with thousands of items, storing sequential integer positions (`0, 1, 2, ... 9999`) causes a **cascading re-index write amplification** problem: dragging item `9999` to index `0` requires updating all 10,000 records in the database.

### O(1) Ordering Data Models

| Strategy | Mechanism | Pros | Cons |
| :--- | :--- | :--- | :--- |
| **Lexicographical Ranking (LexoRank)** | Assigns string ranks (`"a"`, `"b"`). Dropping between `"apple"` and `"banana"` computes midpoint string `"apricot"`. | Arbitrary precision, true **O(1)** single-row update, native SQL `ORDER BY rank ASC`. | Strings grow longer after many tight insertions at the exact same spot; requires rare background rebalancing. |
| **Fractional Indexing** | Assigns float weights (`1.0`, `2.0` -> midpoint `1.5` -> `1.25`). | Simple client math (`(prev + next) / 2`). | Hits IEEE-754 float precision exhaustion ($2^{-52}$) after ~50 consecutive insertions at the same slot unless combined with base-62 string encoding. |
| **Sparse Integer Indexing** | Leaves wide integer gaps (`1000`, `2000`, `3000` -> midpoint `1500`). | Fast integer sorting. | Exhausts gap space after $\log_2(\text{gap})$ insertions (~10 drops for gap 1000), forcing a cascade rebalance. |

### End-to-End Reordering Architecture

```text
[User Drops Card between Item A (rank "B") and Item C (rank "D")]
       │
       ├──> 1. Compute LexoRank Midpoint locally in O(1): newRank = midpoint("B", "D") -> "C"
       ├──> 2. Optimistically mutate local state / Virtualized List (Zero CLS layout shift)
       └──> 3. Fire Debounced PATCH /api/items/:id { rank: "C" }
                 │
                 ├── (200 OK)  ──> Keep optimistic rank
                 └── (Error)   ──> Rollback to pre-drag snapshot + Toast notification
```

```tsx
// Lightweight Base-62 Lexicographical Midpoint Generator
const ALPHABET = '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz';

export function computeLexoMidpoint(prev = '', next = ''): string {
  let result = '';
  let i = 0;

  while (true) {
    const prevCode = i < prev.length ? ALPHABET.indexOf(prev[i]) : 0;
    const nextCode = i < next.length ? ALPHABET.indexOf(next[i]) : ALPHABET.length - 1;

    if (prevCode === nextCode) {
      result += ALPHABET[prevCode];
      i++;
      continue;
    }

    const mid = Math.floor((prevCode + nextCode) / 2);
    if (mid !== prevCode) {
      return result + ALPHABET[mid];
    }

    result += ALPHABET[prevCode];
    i++;
  }
}
```

When combining drag-and-drop with virtualization (`@dnd-kit/core` + [`@tanstack/react-virtual`](https://tanstack.com/virtual/latest/docs/introduction)), keep the actively dragged item mounted or render a `<DragOverlay>` portal so scrolling the virtual list does not unmount the dragged DOM node mid-gesture. Also provide single-pointer or keyboard move controls to satisfy [WCAG 2.2 Dragging Movements (2.5.7)](https://www.w3.org/WAI/standards-guidelines/wcag/new-in-22/).

## 4. Data-Heavy Frontend Apps & [Web Workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API)

### Why Debouncing Alone Does Not Fix Main-Thread Blocking

The browser's **main thread** handles user input events, React reconciliation, layout, paint, and `requestAnimationFrame`. When a user searches or filters 50,000 log lines on the main thread:

```text
[User] ──(Types filter)──> [React Main Thread]
                                  │
                           (Debounce 300ms)
                                  │
                           [analyzeLogs(50,000 items) ~ 400ms CPU]
                           ╔═════════════════════════════════════╗
                           ║    MAIN THREAD IS BLOCKED HERE      ║
                           ║ - Clicks & typing feel frozen       ║
                           ║ - requestAnimationFrame cannot run  ║
                           ╚═════════════════════════════════════╝
                                  │
                           [Update UI -> Responsive again]
```

**Debouncing only reduces how often the heavy function runs; once execution starts on the main thread, the UI still freezes for the entire duration of the computation.**

### Offloading CPU-Intensive Work to [Web Workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers)

Use a [Web Worker](https://developer.mozilla.org/en-US/docs/Web/API/Worker) whenever client-side CPU work (log parsing, fuzzy search across 50k rows, CSV/JSON aggregation, cryptographic hashing, image processing) exceeds **50ms**:

1. **Send Heavy Data Once**: Do not `postMessage` a 50,000-item array on every keystroke (structured cloning large arrays is slow). Send the dataset to the worker **once** on load (`type: 'INIT_DATA'`), and send tiny query payloads (`type: 'FILTER_LOGS'`) on keystrokes.
2. **Prevent Race Conditions with `requestId`**: Assign a monotonically increasing `requestId` to every query and ignore worker responses that do not match `latestRequestIdRef.current`.
3. **Use Transferable Objects (`ArrayBuffer`)** when passing binary buffers for zero-copy O(1) memory transfer.

### Production React + Web Worker Pattern

```ts
// logWorker.ts
interface LogEntry { id: string; level: string; message: string }

let cachedLogs: LogEntry[] = [];

self.onmessage = (event: MessageEvent) => {
  const { type, payload, requestId } = event.data;

  if (type === 'INIT_DATA') {
    cachedLogs = payload;
    return;
  }

  if (type === 'FILTER_LOGS') {
    const keyword = payload.keyword.toLowerCase();
    const matched = cachedLogs.filter((log) =>
      log.message.toLowerCase().includes(keyword)
    );
    self.postMessage({ requestId, matchedCount: matched.length, results: matched.slice(0, 500) });
  }
};
```

```tsx
// useLogWorker.ts
import { useEffect, useRef, useState, useCallback } from 'react';

export function useLogWorker(initialLogs: unknown[]) {
  const workerRef = useRef<Worker | null>(null);
  const latestRequestIdRef = useRef(0);
  const [results, setResults] = useState<unknown[]>([]);
  const [isComputing, setIsComputing] = useState(false);

  useEffect(() => {
    const worker = new Worker(new URL('./logWorker.ts', import.meta.url), {
      type: 'module',
    });
    workerRef.current = worker;

    // Send dataset once!
    worker.postMessage({ type: 'INIT_DATA', payload: initialLogs });

    worker.onmessage = (event) => {
      const { requestId, results: workerResults } = event.data;
      // Guard against stale out-of-order responses
      if (requestId !== latestRequestIdRef.current) return;
      setResults(workerResults);
      setIsComputing(false);
    };

    return () => worker.terminate();
  }, [initialLogs]);

  const runFilter = useCallback((keyword: string) => {
    if (!workerRef.current) return;
    const nextRequestId = ++latestRequestIdRef.current;
    setIsComputing(true);
    workerRef.current.postMessage({
      type: 'FILTER_LOGS',
      requestId: nextRequestId,
      payload: { keyword },
    });
  }, []);

  return { results, isComputing, runFilter };
}
```
