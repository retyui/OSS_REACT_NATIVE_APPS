# perf: stop identity churn and oversubscription across chat/inbox stores
- **Repository:** keybase/client
- **PR Link:** <https://github.com/keybase/client/pull/29411>
- **Merged Date:** 2026-07-13
- **Category:** Rendering

## Problem
Profiling chat send, message receive (while in the inbox) and thread scrollback with the React DevTools profiler showed stores handing out fresh object identities for unchanged data, and components subscribing to whole objects/maps while reading one field. Each store write woke large subscriber sets. Reported dev-build React commit times: sending a message 52 ms to 40 ms (5 whole-screen fan-outs removed); receiving while in the inbox 98 ms to about 25 ms. Scrollback showed zero app subscription wakes during a 100-message ingest.

## Solution
- Producer side: reuse prior objects when deep-equal (conversation metas, inbox layout rows, notification team sets, teams list), and skip store writes when nothing changed.
- Consumer side: add narrow selectors (`useConversationMetaSelector`, typed field picks for inbox rows), migrate always-mounted thread surfaces to them, and make callbacks-only hooks read the store lazily at call time instead of subscribing.
- Compare against committed state rather than the immer draft (lodash `isEqual` breaks on immer 11 drafts).

## Code Pattern (Before vs. After)
```tsx
// Before: callbacks-only hook subscribed to the thread
const {explodingMode, messageMap, messageOrdinals} = useConversationThreadSelector(
  C.useShallow(s => ({
    explodingMode: s.explodingMode,
    messageMap: s.messageMap,
    messageOrdinals: s.messageOrdinals,
  }))
)
const meta = useThreadMeta(C.useShallow(m => ({tlfname: m.tlfname})))
const clientPrev = getClientPrevFromThread(messageMap, messageOrdinals)

// After: read lazily when the callback runs
const threadStore = useConversationThreadStore()
const getTlfName = () => useInboxMetadataState.getState().metas.get(conversationIDKey)?.tlfname ?? ''
const getClientPrev = () => {
  const {messageMap, messageOrdinals} = threadStore.getState()
  return getClientPrevFromThread(messageMap, messageOrdinals)
}
```

```tsx
// Before: every incoming layout replaced all rows
if (!isEqual(get().layout, layout)) {
  s.layout = T.castDraft(layout)
}

// After: keep identity of unchanged rows
const prev = get().layout
const changed = !isEqual(prev, layout)
set(s => {
  if (changed) {
    s.layout = T.castDraft(recycleLayoutRows(prev, layout))
  }
})
```

## Takeaway
Keep object identity stable for unchanged data at the store, and subscribe to the narrowest field; read store state lazily in hooks that only expose callbacks.
