# perf(query-cache): drop app-wide prefetching and fix the cache defaults
- **Repository:** CherryHQ/cherry-studio-app
- **PR Link:** <https://github.com/CherryHQ/cherry-studio-app/pull/494>
- **Merged Date:** 2026-08-07
- **Category:** Networking (+ Lists)

## Problem
Callers of the app's `useQuery` / `useInfiniteQuery` wrappers that never set `staleTime` or `retry` forwarded them as `undefined`. React Query merges options as `{...clientDefaults, ...options}`, so an explicit `undefined` overwrites the client default and every query stayed permanently stale and refetched on every mount. Four prefetch sites (model picker, topic list, AI usage grouping, 85%-scroll older-message preload) had been added to compensate. The PR also reports that reopening a scrolled-back topic re-fetched every loaded page of the infinite query: 8 requests for a 7-page topic. After the change, topic re-entry costs 0 message queries inside the stale window (verified on a simulator).

## Solution
- Add `callerQueryOverrides()` so `retry` and `staleTime` are only forwarded when a caller actually set them. Unset callers inherit the client defaults.
- Delete all four prefetch sites and their helpers, such as `usePrefetchModelPickerData` and the scroll-near-start handler. Nothing is fetched before the user asks for it.
- Give message windows a 5 minute `staleTime`. Freshness comes from `invalidate-topic-messages`.
- `useProviders` always queries the unfiltered `/providers` list and filters in memory, so there is one shared cache entry instead of two.
- Provider invalidation keys drop the trailing params object, because `buildQueryKey` drops empty query objects and the old keys matched nothing.
- The provider list screen moves from `ScrollView` + `map` to `LegendList`.

## Code Pattern (Before vs. After)
```ts
// Before
const result = useTanstackQuery({
  // ...
  queryKey: buildQueryKey(resolvedPath, query),
  retry: options?.retry,          // undefined overrides the client default
  staleTime: options?.staleTime,  // undefined overrides the client default
});

export function useProviders(query: { enabled?: boolean } = {}) {
  const providersQuery = useQuery('/providers', { query }); // second cache entry per filter
  // ...
}

// After
function callerQueryOverrides(options?: { retry?: boolean | number; staleTime?: number }) {
  return {
    ...(options?.retry === undefined ? {} : { retry: options.retry }),
    ...(options?.staleTime === undefined ? {} : { staleTime: options.staleTime }),
  };
}

const result = useTanstackQuery({
  // ...
  queryKey: buildQueryKey(resolvedPath, query),
  ...callerQueryOverrides(options),
});

export function useProviders(query: { enabled?: boolean } = {}) {
  const providersQuery = useQuery('/providers'); // one shared cache entry
  const { enabled } = query;
  const providers = useMemo(
    () => (enabled === undefined ? data : data.filter((provider) => provider.isEnabled === enabled)),
    [data, enabled],
  );
  // ...
}
```

## Takeaway
Never pass `staleTime: undefined` or `retry: undefined` into React Query options, because spreading an explicit `undefined` replaces the client default. Fix defaults first, and drop prefetching that only compensated for them.
