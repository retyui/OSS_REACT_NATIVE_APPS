# perf: fix chat re-render hotspots, drop react-compiler bailouts
- **Repository:** keybase/client
- **PR Link:** <https://github.com/keybase/client/pull/29299>
- **Merged Date:** 2026-06-10
- **Category:** Rendering (+ Lists)

## Problem
Chat thread rows re-rendered on every highlight change because action-only consumers shared one context with the changing state. Separately, the React Compiler bailed out of 81 components/hooks, notably the markdown component (an inline `try/catch` around parsing), so markdown was re-parsed on every text row render. The PR gives no timing numbers.

## Solution
- Split `ConversationCenterContext` into a state context (centered ordinal, highlight) and a stable actions context (`centerOnMessage`, `clearCenter`, `jumpToRecent`); per-row consumers that only dispatch use `useConversationCenterActions`.
- Subscribe to a derived `isEditing` boolean instead of the raw editing ordinal, replace a per-row team-members hook with a context-only role lookup, and compute separator labels only when shown.
- Move `try/catch/finally`, `??=`, getters, dynamic imports and render-time `require`d hooks out of component bodies so the compiler can memoize them; vendored zoom-toolkit opts out with `'use no memo'`. Result: 0 bailouts repo-wide.
- Add `yarn lint:bailouts` to guard against new bailouts (ESLint cannot report them).

## Code Pattern (Before vs. After)
```tsx
// Before
const CenterContext = React.createContext<CenterContextType>({
  centerOnMessage: missingContext,
  centeredHighlightOrdinal: undefined,
  centeredOrdinal: undefined,
  clearCenter: missingContext,
  hasCenter: false,
  jumpToRecent: missingContext,
})
export const useConversationCenter = () => React.useContext(CenterContext)

// After
const CenterStateContext = React.createContext<CenterStateContextType>({/* ordinals, hasCenter */})
const CenterActionsContext = React.createContext<CenterActionsContextType>({/* stable callbacks */})
export const useConversationCenter = () => React.useContext(CenterStateContext)
export const useConversationCenterActions = () => React.useContext(CenterActionsContext)
```

```tsx
// Before: try/catch inline in the component (compiler bails out)
function SimpleMarkdownComponent(p: Props) {
  let parseTree: Array<SM.SingleASTNode>
  let output: React.ReactNode
  try {
    parseTree = parseMarkdown(children ?? '', {isMobile: isMobile, messageType})
    // ...
  } catch (e) { /* ... */ return <ErrorComponent>{children}</ErrorComponent> }

// After: parsing lives in a plain function outside the component
const renderMarkdown = (children, messageType, state, kindOptions) => {
  try { /* ... */ return {output, parseFailed: false} }
  catch (e) { return {output: null, parseFailed: true} }
}
function SimpleMarkdownComponent(p: Props) {
  const {output, parseFailed} = renderMarkdown(children, messageType, state, {/* ... */})
  if (parseFailed) return <ErrorComponent>{children}</ErrorComponent>
```

## Takeaway
Split context into state and actions halves, and keep constructs the React Compiler cannot handle (try/catch, getters) out of component bodies; add a lint check so bailouts do not creep back.
