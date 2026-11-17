# [Perf - part 3] Stop every dialog control in the entire app rerendering when opening a dialog
- **Repository:** bluesky-social/social-app
- **PR Link:** <https://github.com/bluesky-social/social-app/pull/8815>
- **Merged Date:** 2025-08-13
- **Category:** Rendering

## Problem
`fullyExpandedCount` lived inside the dialog control context, so every consumer of that context, meaning every `useDialogControl` call (several per post), re-rendered whenever the count changed, e.g. when the composer opens on iOS or a full-height dialog opens. In the profiler this commit took 180 ms in the simulator, versus about 30 ms for rendering the composer itself. After the change only the composer render spike remains.

## Solution
- Remove `fullyExpandedCount` from the memoized control context value, which now holds only stable callbacks.
- Put the count in its own `DialogFullyExpandedCountContext` with a `useDialogFullyExpandedCountContext` hook.
- Only the shell (which uses it for status bar colour on iOS) subscribes, so only one component re-renders when it changes.

## Code Pattern (Before vs. After)
```tsx
// Before
interface IDialogControlContext {
  closeAllDialogs(): boolean
  setDialogIsOpen(id: string, isOpen: boolean): void
  fullyExpandedCount: number
  setFullyExpandedCount: React.Dispatch<React.SetStateAction<number>>
}
// ...
const controls = React.useMemo(
  () => ({closeAllDialogs, setDialogIsOpen, fullyExpandedCount, setFullyExpandedCount}),
  [closeAllDialogs, setDialogIsOpen, fullyExpandedCount, setFullyExpandedCount],
)
// Shell
const {fullyExpandedCount} = useDialogStateControlContext()

// After
const DialogFullyExpandedCountContext = React.createContext<number>(0)
// ...
const controls = React.useMemo(
  () => ({closeAllDialogs, setDialogIsOpen, setFullyExpandedCount}),
  [closeAllDialogs, setDialogIsOpen, setFullyExpandedCount],
)
// ...
<DialogFullyExpandedCountContext.Provider value={fullyExpandedCount}>
  <GlobalDialogsProvider>{children}</GlobalDialogsProvider>
</DialogFullyExpandedCountContext.Provider>
// Shell
const fullyExpandedCount = useDialogFullyExpandedCountContext()
```

## Takeaway
Never put frequently changing values in a context that many components read only for stable callbacks; split them into separate contexts.
