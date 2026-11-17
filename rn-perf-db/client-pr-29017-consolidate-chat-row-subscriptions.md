# optimize chat thread
- **Repository:** keybase/client
- **PR Link:** <https://github.com/keybase/client/pull/29017>
- **Merged Date:** 2026-03-17
- **Category:** Lists (+ Rendering)

## Problem
Each chat message row (`WrapperMessage`) had about six separate store subscriptions: the wrapper plus the edited marker, text bottom, reply, text type and emoji row each called `useChatContext` on the message map. Every message change woke several selectors per row, and the separator component added two more subscription hooks and a deep component tree. The PR gives no measured numbers; it adds Maestro scroll perf flows for later measurement.

## Solution
- Compute `hasBeenEdited`, `hasCoinFlip`, `hasUnfurlList`, `hasUnfurlPrompts`, `textType` and `showReplyTo` once in `useMessageData` (one `useShallow` selector) and pass them to children as props (6 subscriptions down to 1 primary plus 1 conditional).
- `EmojiRow` takes `messageType`/`hasUnfurls` as optional props; when both are given its selector returns constants, so it never triggers a re-render.
- Merge two separator hooks into one `useSeparatorData`, hoist duplicated profile/tracker hooks, replace `ConnectedUsernames` with plain text and inline `LeftSide`/`TopSide` to flatten the tree.
- List tuning: drop FlashList-only `removeClippedSubviews`/`drawDistance`, set `estimatedItemSize` 100 to 72, and improve `getItemType` (recycle type first, isolate only the last text/attachment as `:pending`).

## Code Pattern (Before vs. After)
```tsx
// Before (text/reply.tsx)
export const useReply = (ordinal: T.Chat.Ordinal) => {
  const showReplyTo = Chat.useChatContext(s => {
    const m = s.messageMap.get(ordinal)
    return m?.type === 'text' ? !!m.replyTo : false
  })
  return showReplyTo ? <Reply /> : null
}

// After
export const useReply = (showReplyTo: boolean) => {
  return showReplyTo ? <Reply /> : null
}
// wrapper.tsx: read once, pass down
const {hasCoinFlip, hasUnfurlList, hasUnfurlPrompts, textType, showReplyTo} = messageData
const bottomChildren = useBottom({hasCoinFlip, hasUnfurlList, hasUnfurlPrompts})
const reply = useReply(showReplyTo)
```

```tsx
// Before (list item type)
if (numOrdinals - 1 === idx) {
  return 'sent'
}
return recycleTypeRef.current.get(ordinal) ?? messageTypeMap.get(ordinal) ?? 'text'

// After
const recycled = recycleTypeRef.current.get(ordinal)
if (recycled) return recycled
const baseType = messageTypeMap.get(ordinal) ?? 'text'
if (numOrdinals - 1 === idx && (baseType === 'text' || baseType === 'attachment')) {
  return `${baseType}:pending`
}
return baseType
```

## Takeaway
In virtualized rows, have one selector per row compute derived flags and pass them down as props instead of letting each child subscribe separately.
