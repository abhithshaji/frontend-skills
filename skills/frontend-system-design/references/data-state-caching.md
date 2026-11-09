# Data, State Architecture, Caching, Normalization, and Schema Safety

This reference covers client-side state taxonomy, multi-layer caching, relational JSON normalization, optimistic UI mutations, and client storage schema drift prevention in [React](https://react.dev/reference/react) applications.

## 1. Frontend State Taxonomy and Colocation

Before reaching for a global store, classify every piece of data into one of five distinct state categories. Mixing these categories causes unnecessary re-renders, stale cache bugs, and bloated boilerplate.

| State Category | Examples | Recommended Tooling | Source of Truth |
| :--- | :--- | :--- | :--- |
| **Server / Remote Cache** | User profile, product catalog, comments | [TanStack Query](https://tanstack.com/query/latest/docs/framework/react/guides/optimistic-updates), SWR, Apollo, [RSC](https://nextjs.org/docs/app/getting-started/server-and-client-components) | Remote Backend Database |
| **URL / Router State** | Search query, active filters, sort order, page/cursor, open modal ID | React Router `useSearchParams`, `nuqs` | Browser URL Bar (Shareable) |
| **Local UI State** | Dropdown open/closed, hover state, active accordion | `useState`, `useReducer` ([React Hooks](https://react.dev/reference/react/hooks)) | Component Instance |
| **Shared Client State** | Theme, sidebar collapsed, multi-step wizard draft, canvas state | [Zustand](https://github.com/pmndrs/zustand), [Redux Toolkit](https://redux.js.org/toolkit/), React Context | Client Memory / Persisted Store |
| **Ephemeral / Transient** | Mouse coordinates, scroll offset, animation timers, uncontrolled inputs | `useRef`, React Hook Form, DOM | Mutable Ref / Native DOM |

## 2. Multi-Layer Frontend Caching Architecture

The frontend is only a temporal snapshot; the server is the source of truth. Effective caching balances **speed**, **freshness**, **correctness**, and **invalidation** across three layers:

```text
[ React Component ]
        │
        ▼
1. In-Memory App Cache (TanStack Query / SWR / React.cache)
   - Instant synchronous reads if data is fresh (within staleTime)
   - Deduplicates concurrent identical requests across components
        │ (Cache miss or stale revalidation)
        ▼
2. Browser HTTP Cache & Service Worker Cache API
   - Cache-Control: max-age, stale-while-revalidate, immutable
   - Conditional requests via ETag / If-None-Match (304 Not Modified)
        │
        ▼
3. Edge CDN & Origin Server
```

### The 4 Questions for Cache Tuning (`staleTime` vs `gcTime`)

When configuring [TanStack Query](https://tanstack.com/query/latest/docs/framework/react/guides/optimistic-updates), distinguish between:

- **`staleTime`**: How long fetched data is considered fresh. While fresh, mounting components read from cache with zero network requests.
- **`gcTime`** (formerly `cacheTime`): How long inactive query data (zero mounted observers) stays in memory before garbage collection.

Use these 4 architectural questions to set `staleTime`:

1. **How fresh does the data need to be?** Low-volatility data (user profile, static config, product specifications) can use `staleTime: 5 * 60 * 1000` (5 minutes) or `Infinity`. Live stock prices or seat availability require `staleTime: 0`.
2. **How expensive is the fetch?** Lightweight payloads can tolerate shorter `staleTime`; expensive aggregations should be cached longer and invalidated explicitly on mutation.
3. **What does the user expect based on their action?** Viewing a tab expects instant cached rendering (`staleTime > 0`), whereas clicking an explicit "Refresh" button or submitting a form expects an immediate refetch (`queryClient.invalidateQueries()`).
4. **What is the blast radius if the data is momentarily stale?** A slightly stale article view count is harmless; a stale checkout price or inventory count causes transaction failures and must always be verified server-side.

### Mutation Synchronization Strategies

When mutating server data, synchronize the client cache using one of three patterns:

1. **Invalidate Associated Queries**: Call `queryClient.invalidateQueries({ queryKey: ['posts'] })` in `onSuccess`. Safest when the backend computes complex side effects, sorting, or aggregations.
2. **Direct Cache Update from Mutation Response**: If the `POST`/`PATCH` endpoint returns the updated canonical entity, call `queryClient.setQueryData(['posts', updated.id], updated)` to save a network round-trip.
3. **Optimistic Updates**: Update the UI immediately before the network responds, and roll back if the server rejects the mutation (detailed in Section 4).

## 3. State Normalization (Flattening Relational Data)

Deeply nested API responses cause three systemic problems in React:

1. **Data Duplication**: The same author object repeated across 50 posts can become inconsistent if updated in one place.
2. **O(N) Nested Traversal**: Updating a deeply nested comment requires complex nested array `.map()` operations.
3. **Cascading Re-renders**: Cloning parent arrays/objects changes references up the tree, forcing unaffected sibling components to re-render.

**Normalization** restructures nested JSON into flat relational lookup tables keyed by ID (similar to an RDBMS), guaranteeing **O(1) reads and writes** and isolated component subscriptions.

### Before vs After Transformation

```json
// AVOID: Deeply Nested Structure
[
  {
    "id": "post-1",
    "title": "React System Design",
    "author": { "id": "user-99", "name": "Alex" },
    "comments": [{ "id": "c-101", "body": "Great guide!" }]
  }
]

// PREFER: Normalized Relational Structure
{
  "result": ["post-1"],
  "entities": {
    "posts": {
      "post-1": {
        "id": "post-1",
        "title": "React System Design",
        "authorId": "user-99",
        "commentIds": ["c-101"]
      }
    },
    "users": {
      "user-99": { "id": "user-99", "name": "Alex" }
    },
    "comments": {
      "c-101": { "id": "c-101", "body": "Great guide!" }
    }
  }
}
```

### Pattern A: Normalized Store with [Zustand](https://github.com/pmndrs/zustand) (Selective O(1) Subscriptions)

By passing only `postId` to child rows and subscribing to `state.posts[postId]`, updating one post or user never re-renders unrelated posts in the list:

```tsx
import { create } from 'zustand';

interface User { id: string; name: string }
interface Post { id: string; title: string; authorId: string }

interface NormalizedFeedState {
  postIds: string[];
  posts: Record<string, Post>;
  users: Record<string, User>;
  ingestFeed: (rawPosts: Array<{ id: string; title: string; author: User }>) => void;
  renameUser: (userId: string, newName: string) => void;
}

export const useFeedStore = create<NormalizedFeedState>((set) => ({
  postIds: [],
  posts: {},
  users: {},
  ingestFeed: (rawPosts) => {
    const postIds: string[] = [];
    const posts: Record<string, Post> = {};
    const users: Record<string, User> = {};

    for (const item of rawPosts) {
      postIds.push(item.id);
      users[item.author.id] = item.author;
      posts[item.id] = { id: item.id, title: item.title, authorId: item.author.id };
    }
    set({ postIds, posts, users });
  },
  // O(1) single-key update!
  renameUser: (userId, newName) =>
    set((state) => ({
      users: {
        ...state.users,
        [userId]: { ...state.users[userId], name: newName },
      },
    })),
}));

// Child component subscribes ONLY to its specific entity ID
export function PostRow({ postId }: { postId: string }) {
  const post = useFeedStore((s) => s.posts[postId]);
  const authorName = useFeedStore((s) => (post ? s.users[post.authorId]?.name : ''));

  if (!post) return null;
  return (
    <article>
      <h3>{post.title}</h3>
      <span>By {authorName}</span>
    </article>
  );
}
```

### Pattern B: Redux Toolkit [`createEntityAdapter`](https://redux.js.org/toolkit/api/createEntityAdapter) or `normalizr`

- When using [Redux Toolkit](https://redux.js.org/toolkit/), call [`createEntityAdapter()`](https://redux.js.org/toolkit/api/createEntityAdapter) to automatically generate `{ ids: [], entities: {} }` state shapes and O(1) CRUD reducers (`setAll`, `updateOne`, `upsertMany`).
- For deeply recursive schemas, normalize payloads at the API boundary before ingesting into client stores.

## 4. Optimistic Updates and Reconciliation

Optimistic updates reflect user actions (likes, bookmarks, todo toggles, kanban moves, chat messages) in the UI **immediately**, firing the network request asynchronously in the background and reconciling when the server responds.

```text
[User Action: Click Like]
       │
       ├──> 1. Immediate UI Update: optimisticState = currentState + 1
       └──> 2. Async Network Request (inside startTransition / onMutate)
                 │
                 ├── (200 OK)  ──> Commit real state (setLikes / invalidate)
                 └── (Error)   ──> Automatic rollback to committed state + Show Toast
```

### Pattern A: React 19 [`useOptimistic`](https://react.dev/reference/react/useOptimistic) + `startTransition`

**Critical Rule**: In React 19, the updater returned by [`useOptimistic`](https://react.dev/reference/react/useOptimistic) **must** be called inside `startTransition` (or a `<form action={...}>`). When the async transition finishes, React automatically discards the temporary optimistic projection and falls back to the committed `currentState`. If the request succeeded, updating `currentState` inside the transition commits the new value seamlessly; if it threw an error, leaving `currentState` untouched automatically rolls back the UI without needing a no-op state setter.

```tsx
import { useOptimistic, useState, useTransition } from 'react';

async function toggleLikeOnServer(postId: string, nextLiked: boolean) {
  const res = await fetch(`/api/posts/${postId}/like`, {
    method: 'POST',
    body: JSON.stringify({ liked: nextLiked }),
  });
  if (!res.ok) throw new Error('Failed to persist like');
  return res.json();
}

export function LikeButton({
  postId,
  initialCount,
  initialLiked,
}: {
  postId: string;
  initialCount: number;
  initialLiked: boolean;
}) {
  // 1. Committed source of truth
  const [committed, setCommitted] = useState({
    count: initialCount,
    liked: initialLiked,
  });
  const [errorMsg, setErrorMsg] = useState<string | null>(null);
  const [, startTransition] = useTransition();

  // 2. Pure optimistic projection: (currentState, optimisticInput) => nextOptimisticState
  const [optimistic, setOptimistic] = useOptimistic(
    committed,
    (state, nextLiked: boolean) => ({
      liked: nextLiked,
      count: state.count + (nextLiked ? 1 : -1),
    })
  );

  const handleToggle = () => {
    const nextLiked = !optimistic.liked;
    setErrorMsg(null);

    startTransition(async () => {
      // Immediately project the optimistic state for the duration of this transition
      setOptimistic(nextLiked);

      try {
        await toggleLikeOnServer(postId, nextLiked);
        // Commit the real state so when transition ends, committed matches optimistic
        setCommitted((prev) => ({
          liked: nextLiked,
          count: prev.count + (nextLiked ? 1 : -1),
        }));
      } catch {
        // Do NOT update committed state; React automatically reverts optimistic -> committed
        setErrorMsg('Could not save like. Reverted.');
      }
    });
  };

  return (
    <div>
      <button
        type="button"
        onClick={handleToggle}
        aria-pressed={optimistic.liked}
      >
        {optimistic.liked ? 'Liked' : 'Like'} ({optimistic.count})
      </button>
      {errorMsg && <span role="alert" className="text-red-600 text-sm">{errorMsg}</span>}
    </div>
  );
}
```

### Pattern B: [TanStack Query Optimistic Updates](https://tanstack.com/query/latest/docs/framework/react/guides/optimistic-updates) with Snapshot Rollback

When data lives in TanStack Query cache, always cancel in-flight refetches in `onMutate` so an older background GET does not overwrite your optimistic update:

```tsx
import { useMutation, useQueryClient } from '@tanstack/react-query';

export function useOptimisticTodoToggle() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, completed }: { id: string; completed: boolean }) =>
      fetch(`/api/todos/${id}`, {
        method: 'PATCH',
        body: JSON.stringify({ completed }),
      }).then((r) => {
        if (!r.ok) throw new Error('Failed');
        return r.json();
      }),

    onMutate: async ({ id, completed }) => {
      // 1. Cancel outgoing refetches so they do not overwrite our optimistic update
      await queryClient.cancelQueries({ queryKey: ['todos'] });

      // 2. Snapshot the previous cache value for rollback
      const previousTodos = queryClient.getQueryData(['todos']);

      // 3. Optimistically update the cache
      queryClient.setQueryData(['todos'], (old: Array<{ id: string; completed: boolean }> = []) =>
        old.map((t) => (t.id === id ? { ...t, completed } : t))
      );

      return { previousTodos };
    },

    onError: (_err, _variables, context) => {
      // 4. Rollback to snapshot on failure
      if (context?.previousTodos) {
        queryClient.setQueryData(['todos'], context.previousTodos);
      }
    },

    onSettled: () => {
      // 5. Always refetch after error or success to ensure server reconciliation
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
  });
}
```

## 5. Client Schema Drift and Storage Safety

Unlike backend databases where migrations run once under your control, client-side storage ([`localStorage`](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage), `sessionStorage`, [`IndexedDB`](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API)) persists indefinitely across user browsers. **Client Schema Drift** creates two production failure modes:

1. **Forward Compatibility Crash**: You deploy `v2` of your React app expecting `filters` in `localStorage` to be a JSON object `{ tags: string[] }`, but returning users still have `v1` data stored as a comma-separated string `"react,vite"`, crashing the app on boot (`TypeError: filters.tags.map is not a function`).
2. **Rollback Failure Crash**: You deploy `v2`, which writes the new format into `localStorage`. Five minutes later, you roll back the release to `v1` due to a bug. Now `v1` code crashes for every user who visited during those five minutes because `v1` cannot parse the `v2` schema left behind in their browser.

### Recommended Defense Patterns

1. **Defensive Reading + Runtime Validation (Zod)**: Treat `localStorage` and `IndexedDB` as untrusted external input. Always wrap `JSON.parse` in `try/catch` and validate with a schema parser; fall back to clean defaults on failure.
2. **Envelope Version Tagging (Forward Migrations)**: Store `{ version: number, data: T }` and run sequential migration functions (`v1 -> v2 -> v3`) when reading older envelopes.
3. **Storage Key Versioning (Rollback Immunity)**: When making breaking schema changes, increment the storage key name itself (e.g., `app:filters:v1` -> `app:filters:v2`). If a deployment is rolled back to `v1`, `v1` reads `app:filters:v1` untouched and never collides with `app:filters:v2`.

### Production Versioned Storage Helper

```tsx
import { z } from 'zod';

const FilterSchemaV2 = z.object({
  version: z.literal(2),
  data: z.object({
    tags: z.array(z.string()),
    sortBy: z.enum(['date', 'popularity']),
  }),
});

type FilterDataV2 = z.infer<typeof FilterSchemaV2>['data'];

const STORAGE_KEY = 'app:filters:v2'; // Key versioning protects rollbacks
const DEFAULT_FILTERS: FilterDataV2 = { tags: [], sortBy: 'date' };

export function readSafeFilters(): FilterDataV2 {
  if (typeof window === 'undefined') return DEFAULT_FILTERS;

  try {
    const raw = window.localStorage.getItem(STORAGE_KEY);
    if (!raw) return DEFAULT_FILTERS;

    const parsed = JSON.parse(raw);

    // Optional: Migrate from v1 envelope if using same key
    if (parsed?.version === 1 && typeof parsed.data === 'string') {
      const migrated: FilterDataV2 = {
        tags: parsed.data.split(',').filter(Boolean),
        sortBy: 'date',
      };
      writeSafeFilters(migrated);
      return migrated;
    }

    const result = FilterSchemaV2.safeParse(parsed);
    if (!result.success) {
      window.localStorage.removeItem(STORAGE_KEY);
      return DEFAULT_FILTERS;
    }
    return result.data.data;
  } catch {
    // Corrupted JSON or storage access blocked (private browsing)
    return DEFAULT_FILTERS;
  }
}

export function writeSafeFilters(data: FilterDataV2): void {
  try {
    window.localStorage.setItem(
      STORAGE_KEY,
      JSON.stringify({ version: 2, data })
    );
  } catch {
    // QuotaExceededError or SecurityError
  }
}
```
