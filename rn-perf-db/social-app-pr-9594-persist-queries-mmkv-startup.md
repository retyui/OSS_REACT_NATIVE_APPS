# Speed up startup by persisting some queries
- **Repository:** bluesky-social/social-app
- **PR Link:** <https://github.com/bluesky-social/social-app/pull/9594>
- **Merged Date:** 2026-02-02
- **Category:** Startup (+ Storage & IO)

## Problem
At launch the app had a waterfall: it fetched preferences and then feed info before it could render the feed skeletons, so users saw a circular spinner first. The PR does not report timings, only before/after screen recordings in which the spinner stage disappears and the app goes straight to skeletons.

## Solution
- Persist the preferences query and the pinned/saved feed-info queries via React Query's persister, so they hydrate from disk and the first fetch stage is skipped.
- Any query whose key starts with the new `PERSISTED_QUERY_ROOT` is dehydrated; those queries use `gcTime: Infinity` so the cache is not discarded.
- Replaced `AsyncStorage` with a per-DID MMKV store (`createPersistedQueryStorage`) behind the async-persister interface. MMKV is synchronous, so it is wrapped in Promises only for API compatibility.
- Added `superjson` to serialise `Date`s and `buster: APP_VERSION` so a new app version invalidates the cache.
- Caveat noted in the PR: persisted cache dates come back as strings, so `usePreferencesQuery` re-hydrates `birthDate` into a `Date`.

## Code Pattern (Before vs. After)
```tsx
// Before
const asyncPersister = createAsyncStoragePersister({
  storage: AsyncStorage,
  key: 'queryClient-' + (currentDid ?? 'logged-out'),
})
// ...
export const preferencesQueryKey = ['getPreferences']
shouldDehydrateQuery: query =>
  STORED_CACHE_QUERY_KEY_ROOTS.includes(String(query.queryKey[0]))

// After
const storage = createPersistedQueryStorage(currentDid ?? 'logged-out')
const asyncPersister = createAsyncStoragePersister({
  storage,
  key: 'queryClient-' + (currentDid ?? 'logged-out'),
})
return {
  persister: asyncPersister,
  dehydrateOptions,
  buster: env.APP_VERSION,
}
// ...
export const preferencesQueryKey = [PERSISTED_QUERY_ROOT, 'getPreferences']
const query = useQuery({
  queryKey: preferencesQueryKey,
  gcTime: PERSISTED_QUERY_GCTIME,
  // ...
})
shouldDehydrateQuery: query => String(query.queryKey[0]) === PERSISTED_QUERY_ROOT
```

## Takeaway
Persist the small, slow-changing queries that gate first render (preferences, config) on a sync store like MMKV, with a long `gcTime` and a version buster.
