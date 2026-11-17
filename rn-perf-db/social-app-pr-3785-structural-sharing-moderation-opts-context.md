# Performance improvements: structural sharing & moderation opts context
- **Repository:** bluesky-social/social-app
- **PR Link:** <https://github.com/bluesky-social/social-app/pull/3785>
- **Merged Date:** 2024-05-01
- **Category:** Rendering

## Problem
The preferences query had `structuralSharing: true`, but the SDK puts a `Date` inside the `getPreferences` response and React Query's comparison does not treat equal dates as equal, so every refetch returned a new object and re-triggered dependent hooks. Separately, `useModerationOpts` was a hook called in many leaf components, each building its own options object and widening dependency trees. The PR reports improvements in the React flamegraph and JS profiler on a web thread screen (screenshots only, no numbers) and says the measurement may not be precise.

## Solution
- Add a `replaceEqualDeep` function (copied from TanStack Query's `utils.ts`) modified to compare `Date` objects by `getTime()`, and pass it as `structuralSharing` for the preferences query so unchanged responses keep the same reference.
- Move moderation options into a `ModerationOptsProvider` context at the app root. It computes the options once with `useMemo`, and `useModerationOpts()` becomes a plain `useContext` read.
- Update all consumers (post feed, thread, notifications, profile cards, search, etc.) to import the new hook.

## Code Pattern (Before vs. After)
```ts
// Before (state/queries/preferences/index.ts)
return useQuery({
  staleTime: STALE.SECONDS.FIFTEEN,
  structuralSharing: true,
  // ...
})
export function useModerationOpts() {
  const override = useContext(moderationOptsOverrideContext)
  const prefs = usePreferencesQuery()
  const opts = useMemo<ModerationOpts | undefined>(() => { /* build per call */ }, [/* ... */])
  return opts
}

// After
return useQuery({
  staleTime: STALE.SECONDS.FIFTEEN,
  structuralSharing: replaceEqualDeep,
  // ...
})
// state/preferences/moderation-opts.tsx
export function useModerationOpts() {
  return useContext(moderationOptsContext)
}
export function Provider({children}: React.PropsWithChildren<{}>) {
  // ...
  const value = useMemo<ModerationOpts | undefined>(() => { /* build once */ },
    [override, userDid, labelDefs, moderationPrefs, hiddenPosts])
  return (
    <moderationOptsContext.Provider value={value}>{children}</moderationOptsContext.Provider>
  )
}
```
```ts
// replaceEqualDeep: Date support added
if (a instanceof Date && b instanceof Date) {
  return a.getTime() === b.getTime() ? a : b
}
```

## Takeaway
Verify that query structural sharing actually returns stable references (Dates and class instances break it), and hoist derived values used by many leaf components into one memoized context.
