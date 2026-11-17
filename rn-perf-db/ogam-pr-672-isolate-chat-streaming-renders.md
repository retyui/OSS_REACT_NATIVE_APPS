# perf(mobile): isolate chat streaming from workspace renders and sync
- **Repository:** off-grid-ai/OGAM
- **PR Link:** <https://github.com/off-grid-ai/OGAM/pull/672>
- **Merged Date:** 2026-09-13
- **Category:** Rendering (+ Storage & IO)

## Problem
While a chat reply streamed, the whole chat state was repeatedly persisted to storage, and screens such as Home and Projects subscribed to the entire chat store and re-rendered on every token. This made Projects laggy during a streaming reply. The PR reports no measured numbers; validation was a manual "no Mobile lag" report plus the test suite.

## Solution
- The persisted storage wrapper (`createHydrationGatedStorage`) takes an `unchanged(previous, next)` comparator and skips `setItem` when the persisted slice (`conversations`, `activeConversationId`) is unchanged, tracking the last written state.
- Home and Projects screens use per-field Zustand selectors instead of destructuring the whole store, so streaming fields no longer re-render them.
- `recentConversations` on Home and per-project chat counts on Projects are memoized on `conversations` (counts computed in one pass instead of a filter per row).
- Small chat drafts and edits stay local to their controls (per the PR description).

## Code Pattern (Before vs. After)
```tsx
// Before
const { projects, deleteProject } = useProjectStore();
const { conversations } = useChatStore();

const getChatCount = (projectId: string) =>
  conversations.filter((c) => c.projectId === projectId).length;

// After
const projects = useProjectStore(state => state.projects);
const deleteProject = useProjectStore(state => state.deleteProject);
const conversations = useChatStore(state => state.conversations);

const chatCounts = useMemo(() => {
  const counts: Record<string, number> = {};
  for (const conversation of conversations) {
    if (conversation.projectId) {
      counts[conversation.projectId] = (counts[conversation.projectId] ?? 0) + 1;
    }
  }
  return counts;
}, [conversations]);
```
```ts
// Before
const chatStorage = createHydrationGatedStorage<PersistedChatState>();

// After
const chatStorage = createHydrationGatedStorage<PersistedChatState>(
  undefined,
  (previous, next) =>
    previous.conversations === next.conversations &&
    previous.activeConversationId === next.activeConversationId,
);
// in setItem:
if (lastWritten !== undefined && unchanged?.(lastWritten, value.state)) {
  return undefined;
}
```

## Takeaway
Use selectors rather than whole-store destructuring, and make persist middleware skip writes when the persisted slice is referentially unchanged, so high-frequency state such as streaming does not hit storage or unrelated screens.
