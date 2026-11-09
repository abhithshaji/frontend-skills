# Real-Time Updates, Streaming UI, OT vs CRDT, and Offline-First Architecture

This reference covers real-time communication protocols, multi-server WebSocket scaling, byte-level NDJSON/SSE streaming UIs, collaborative conflict resolution (OT vs CRDT), and offline-first [React](https://react.dev/reference/react) architectures.

## 1. Real-Time Transport Selection Matrix

Real-time updates keep the React UI synchronized with backend events or peer actions without requiring manual reloads. Choose the transport mechanism based on directionality, latency tolerance, and infrastructure complexity:

| Protocol | Direction | Underlying Transport | Best Use Cases | Trade-offs |
| :--- | :--- | :--- | :--- | :--- |
| **Short Polling** | Client Pull | Repeated HTTP `GET` on timer | Simple status checks, low-frequency dashboards | High header overhead; wasted requests when nothing changed |
| **[Server-Sent Events (SSE)](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)** | Unidirectional (Server -> Client) | Long-lived HTTP (`text/event-stream`) | AI token streaming, live feeds, stock tickers, notifications | Works over standard HTTP/2; auto-reconnects; no binary frames |
| **[WebSockets](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)** | Bidirectional Full-Duplex | Upgraded TCP (`ws://` / `wss://`) | Chat rooms, multiplayer cursors, collaborative editing, gaming | Requires sticky load balancing, custom reconnect/backoff, and Pub/Sub scaling |
| **GraphQL Subscriptions** | Unidirectional Event Stream | WebSockets (`graphql-ws`) or SSE | Apps already standardized on GraphQL normalized caches | Ties real-time schema tightly to GraphQL server infrastructure |

### Scaling WebSockets Across Multiple Servers (Redis Pub/Sub)

With a single backend server, User A and User B connect to the same process in memory. In production, horizontal scaling places multiple stateless [WebSocket](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API) servers behind a load balancer:

```text
[User A Browser] <══ WebSocket ══> [WS Server 1] <──┐
                                                    ├──> [Redis Pub/Sub / Message Queue]
[User B Browser] <══ WebSocket ══> [WS Server 2] <──┘
```

If User A (connected to Server 1) sends a message intended for User B (connected to Server 2), Server 1 publishes the event to a **Redis Pub/Sub** channel or message queue. All subscribed WebSocket servers receive the broadcast and forward the payload to their locally connected client sockets.

### Production WebSocket Hook + TanStack Query Integration

Never replace your entire HTTP cache with raw WebSocket state. Fetch initial snapshots over HTTP (cached by [TanStack Query](https://tanstack.com/query/latest/docs/framework/react/guides/optimistic-updates)), and use WebSockets to push incremental deltas or targeted cache invalidations:

```tsx
import { useEffect, useRef } from 'react';
import { useQueryClient } from '@tanstack/react-query';

export function useRealtimeCacheSync(url: string) {
  const queryClient = useQueryClient();
  const retryCountRef = useRef(0);

  useEffect(() => {
    let ws: WebSocket | null = null;
    let reconnectTimer: ReturnType<typeof setTimeout>;
    let isUnmounted = false;

    function connect() {
      if (isUnmounted) return;
      ws = new WebSocket(url);

      ws.onopen = () => {
        retryCountRef.current = 0; // Reset backoff on healthy connection
      };

      ws.onmessage = (event) => {
        try {
          const msg = JSON.parse(event.data);
          // Mutate TanStack Query cache directly without triggering a network refetch
          if (msg.type === 'MESSAGE_CREATED') {
            queryClient.setQueryData(
              ['messages', msg.channelId],
              (old: Array<{ id: string }> = []) =>
                old.some((m) => m.id === msg.payload.id) ? old : [...old, msg.payload]
            );
          }
        } catch (err) {
          console.error('Malformed socket payload', err);
        }
      };

      ws.onclose = () => {
        if (isUnmounted) return;
        // Exponential backoff with random jitter (prevents thundering herd on server restart)
        const baseDelay = Math.min(30_000, 1000 * 2 ** retryCountRef.current);
        const jitter = Math.random() * 1000;
        retryCountRef.current += 1;
        reconnectTimer = setTimeout(connect, baseDelay + jitter);
      };
    }

    connect();
    return () => {
      isUnmounted = true;
      clearTimeout(reconnectTimer);
      ws?.close();
    };
  }, [url, queryClient]);
}
```

## 2. Streaming UI (Byte-Level NDJSON & ReadableStream Pipeline)

Modern AI interfaces and progressive data loaders do not wait for `await response.json()`. Instead, they process a continuous byte stream using the browser [`ReadableStream` API](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream) and render incremental updates token-by-token.

```text
[Server Stream: NDJSON]
{"type":"meta","model":"gemini"}\n{"type":"delta","text":"Hello"}\n{"type":"done"}\n
        │
        ▼ (Raw Uint8Array byte chunks; may split mid-line or mid-UTF8 character!)
[response.body.getReader()]
        │
        ▼
[TextDecoder('utf-8', { stream: true })]  <-- Preserves multi-byte UTF-8 across chunks
        │
        ▼
[Line Buffer] -> Splits on '\n', keeps trailing incomplete fragment for next chunk
        │
        ▼
[JSON.parse(line)] -> Dispatches 'meta' | 'delta' | 'done' | 'error' to React state
```

### Critical Streaming Implementation Rules

1. **Always pass `{ stream: true }` to `TextDecoder.decode(value, { stream: true })`**: Multi-byte UTF-8 characters (like emojis or non-ASCII scripts) can split across two TCP packets. Without `{ stream: true }`, split characters corrupt into replacement characters.
2. **Maintain a Trailing Line Buffer**: A single network chunk may contain half a JSON object, or three JSON objects at once. Append decoded text to `buffer`, split by `\n`, pop the last element back into `buffer` (as it may be incomplete), and parse all complete lines.
3. **Support Cancellation with `AbortController`**: Always abort the stream on component unmount or when the user clicks "Stop Generating".

### Production NDJSON Streaming Hook

```tsx
import { useState, useRef, useCallback, useEffect } from 'react';

type StreamEvent =
  | { type: 'meta'; requestId: string }
  | { type: 'delta'; text: string }
  | { type: 'done' }
  | { type: 'error'; message: string };

export function useStreamingResponse() {
  const [content, setContent] = useState('');
  const [isStreaming, setIsStreaming] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const abortControllerRef = useRef<AbortController | null>(null);

  const stopStream = useCallback(() => {
    abortControllerRef.current?.abort();
    setIsStreaming(false);
  }, []);

  const startStream = useCallback(async (url: string, body: unknown) => {
    abortControllerRef.current?.abort();
    const controller = new AbortController();
    abortControllerRef.current = controller;

    setIsStreaming(true);
    setContent('');
    setError(null);

    try {
      const response = await fetch(url, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(body),
        signal: controller.signal,
      });

      if (!response.ok || !response.body) {
        throw new Error(`HTTP ${response.status}`);
      }

      const reader = response.body.getReader();
      const decoder = new TextDecoder('utf-8');
      let buffer = '';

      while (true) {
        const { value, done } = await reader.read();
        if (done) break;

        // Decode bytes while preserving split multi-byte sequences
        buffer += decoder.decode(value, { stream: true });
        const lines = buffer.split('\n');

        // Keep the potentially incomplete last segment in the buffer
        buffer = lines.pop() ?? '';

        for (const rawLine of lines) {
          const line = rawLine.trim();
          if (!line) continue;

          const event = JSON.parse(line) as StreamEvent;
          if (event.type === 'delta') {
            setContent((prev) => prev + event.text);
          } else if (event.type === 'error') {
            setError(event.message);
          } else if (event.type === 'done') {
            setIsStreaming(false);
          }
        }
      }
    } catch (err) {
      if ((err as Error).name !== 'AbortError') {
        setError((err as Error).message || 'Stream interrupted');
      }
    } finally {
      setIsStreaming(false);
    }
  }, []);

  useEffect(() => () => abortControllerRef.current?.abort(), []);

  return { content, isStreaming, error, startStream, stopStream };
}
```

## 3. Collaborative Conflict Resolution: OT vs CRDT

When two users concurrently edit the same document (e.g., Google Docs, Notion, Figma), naive **Last-Write-Wins (LWW)** overwrites one user's changes completely. Production collaborative systems use either **Operational Transformation (OT)** or **Conflict-Free Replicated Data Types ([CRDTs](https://docs.yjs.dev/))**.

### Operational Transformation (OT)

- **Mechanism**: Operations are index-based (e.g., `insert("X", index: 0)`). A **centralized server** acts as the single serialization point, transforming incoming operations against already-applied concurrent operations so user intent is preserved.
- **Worked Example**:
  - Initial document: `"Hello"` (indices `0..4`).
  - User A sends `insert("X", 0)`. User B concurrently sends `insert("Y", 0)`.
  - Server applies A's edit first -> Document becomes `"XHello"`.
  - When B's operation arrives, the server transforms B's target index from `0` to `1` to account for A's insertion at `0` -> `"XYHello"` (both edits preserved).
- **Trade-offs**: Lightweight network payloads, but requires a central coordinating server and complex transformation matrices. Poor fit for long offline durations or peer-to-peer sync.

### Conflict-Free Replicated Data Types ([CRDTs](https://docs.yjs.dev/))

- **Mechanism**: Eliminates positional indices entirely. Every character or element is modeled as an immutable node with a **globally unique, deterministically ordered ID** (e.g., `(ClientID, LogicalClock)`), referencing its left/right neighbor IDs:

```text
Document "Hello" as CRDT linked nodes:
(H, id0) ──> (e, id1) ──> (l, id2) ──> (l, id3) ──> (o, id4)

Concurrent Insert at start by User A ('X', idA) and User B ('Y', idB):
Merge rule: Both reference 'start' -> deterministic tie-break by ID (idA < idB)
Result on ALL peers without central coordinator:
[X : idA] ──> [Y : idB] ──> [H : id0] ──> [e : id1] ──> [l : id2] ──> [l : id3] ──> [o : id4]
```

- **Trade-offs**: Moves complexity into the data structure metadata. Guarantees **Strong Eventual Consistency** without a central transformation server, making CRDT libraries like [Yjs](https://docs.yjs.dev/) ideal for **offline-first**, local-first, and peer-to-peer collaborative React apps.

| Dimension | Operational Transformation (OT) | CRDT (e.g., [Yjs](https://docs.yjs.dev/), Automerge) |
| :--- | :--- | :--- |
| **Coordination** | Requires centralized authoritative server | Decentralized; merges deterministically on any node |
| **Addressing Model** | Positional integer indices (`index: 0`) | Globally unique immutable node IDs + causal links |
| **Offline Support** | Fragile over long offline divergences | Native offline-first and P2P convergence |
| **Memory Overhead** | Low (plain strings/JSON on client) | Higher (stores metadata and tombstones for deletions) |

## 4. Offline-First Frontend Architecture

An offline-first architecture decouples the React UI from immediate network availability by routing reads and writes through local storage layers and a background synchronization engine:

```text
[ React UI Components ] <───> [ Client State / TanStack Query (networkMode: 'offlineFirst') ]
                                          │
                 ┌────────────────────────┴────────────────────────┐
                 ▼                                                 ▼
     [ Service Worker Proxy ]                         [ Local Sync & Outbox Engine ]
                 │                                                 │
                 ▼                                                 ▼
     [ Cache API (App Shell / Bundles) ]              [ IndexedDB (Relational Data & Write Queue) ]
                 │                                                 │
                 └────────────────────────┬────────────────────────┘
                                          ▼
                                 ( Network Bridge )
                                          ▼
                                 [ Remote API Server ]
```

### Client-Side Storage Selection Matrix

| Storage Mechanism | Capacity | API Nature | Querying Capability | Best Used For |
| :--- | :--- | :--- | :--- | :--- |
| **Cache API** | 50MB+ (Quota based) | Async (Promise) | Request/Response URL matching | App shell HTML, JS/CSS bundles, fonts, static images |
| **[IndexedDB](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API)** | Up to disk quota (GBs) | Async / Event / Wrapper (`idb`) | Primary keys, secondary indexes, cursors | Structured app data, offline mutation write queues, large blobs |
| **[localStorage](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage)** | ~5MB | **Synchronous** (Blocks main thread!) | Key-value string lookup only | Small UI preferences, theme flags (Never store large JSON or tokens) |

### [Service Worker](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API) Caching Strategies

- **Cache-First**: Serve directly from Cache API; fall back to network only on miss. Ideal for content-hashed static bundles (`app.8f9a2.js`, fonts, immutable images).
- **Network-First (with Cache Fallback)**: Attempt network fetch with a timeout; fall back to cached response if offline. Ideal for frequently updated API feeds where fresh data is preferred.
- **Stale-While-Revalidate (SWR)**: Return cached entry immediately for instant UI rendering, then fire a background network request to update the cache for next time.

### IndexedDB Offline Write Outbox & Connectivity Truth-Checking

When the user mutates data while offline, append an operation with a client-generated **idempotency key (`UUID`)** to an [IndexedDB](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API) `outbox` store and apply the change optimistically to the local read store.

- **Background Sync API**: Register a Service Worker sync tag (`registration.sync.register('flush-outbox')`) so the browser flushes queued writes automatically when connectivity returns, even if the user leaves the page.
- **Never trust `navigator.onLine` alone**: `navigator.onLine === true` only means the device is connected to a router/LAN, not that it has working internet or that your API is reachable (e.g., captive hotel Wi-Fi). Combine `window.addEventListener('online' | 'offline')` with lightweight `/api/health` heartbeat checks and ambient UX indicators ("Viewing offline snapshot - 3 changes pending sync").
