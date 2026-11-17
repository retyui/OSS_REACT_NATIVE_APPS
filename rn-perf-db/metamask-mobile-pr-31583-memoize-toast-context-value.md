# perf: memoize ToastContext provider value to prevent unnecessary re-renders
- **Repository:** MetaMask/metamask-mobile
- **PR Link:** <https://github.com/MetaMask/metamask-mobile/pull/31583>
- **Merged Date:** 2026-06-16
- **Category:** Rendering

## Problem
`ToastContextWrapper` passed an inline object literal `{ toastRef }` as the `ToastContext.Provider` value on every render. Since `Object.is` always fails on a new object, all roughly 52 `ToastContext` consumers re-rendered whenever the wrapper re-rendered, even though `toastRef` (a `useRef`) never changes. The PR reports no before/after measurements.

## Solution
The value is wrapped in `useMemo` with an empty dependency array. A ref's identity is stable by definition, so the memoized object is created once and reused, which removes the fan-out re-renders across consumers.

## Code Pattern (Before vs. After)
```tsx
// Before
return (
  <ToastContext.Provider value={{ toastRef }}>
    {children}
  </ToastContext.Provider>
);

// After
const value = useMemo(() => ({ toastRef }), []);

return (
  <ToastContext.Provider value={value}>{children}</ToastContext.Provider>
);
```

## Takeaway
Always memoize the object passed to a Context provider `value`, especially for widely consumed contexts.
