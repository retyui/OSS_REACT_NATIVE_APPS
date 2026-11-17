# perf(chat): reduce render fanout and share one Expo runtime
- **Repository:** CherryHQ/cherry-studio-app
- **PR Link:** <https://github.com/CherryHQ/cherry-studio-app/pull/1160>
- **Merged Date:** 2026-10-08
- **Category:** Rendering (+ Storage & IO)

## Problem
Typing and streaming rebuilt the chat input and unchanged message parts: each remote update re-projected every part, the tool list and the composer actions, and drafts were serialized on every keystroke. Separately, pnpm peer variants gave `@expo/ui` a second Expo runtime, which broke asynchronous bundle loading with an uninitialized HMR client. The PR states that runtime correctness was verified but that no quantitative render/FPS improvement is claimed, so there are no before/after numbers.

## Solution
- Add a `RemoteMessageProjectionCache`: a `WeakMap` keyed by the remote part object stores its projected part and tool. A streamed update replaces only changed parts, so the others keep their identity. The tool list also keeps its previous array when no element changed. Completed text parts leave streaming mode on their own.
- `ComposerSurface` keeps `draft`, `attachments`, `onSend` and `onStop` in a ref and exposes stable `handleSend` / `handleStop` callbacks and memoized labels, so the Composer's actions do not change as the user types.
- `useRemoteDraftPersistence` debounces draft writes by 500 ms and flushes on key change, unmount and backgrounding. Drafts are read without subscribing, so typing does not re-render the chat screen.
- Metro resolves `expo` and `expo/*` from the project root so only one Expo/HMR runtime is bundled.

## Code Pattern (Before vs. After)
```ts
// Before: every update re-projects all parts and rebuilds the tool array
for (const part of message.parts) {
  if (part.kind === 'text' || part.kind === 'reasoning') {
    parts.push({ type: part.kind, text: part.text, state: message.state === 'streaming' ? 'streaming' : 'done' });
  }
  // ...
}
return { /* ... */ tools };

// After: unchanged parts reuse the cached projection
let projected = cache?.parts.get(part);
if (projected?.streaming !== streaming) {
  projected = { streaming, ...projectRemotePart(part, streaming, resource) };
  cache?.parts.set(part, projected);
}
parts.push(projected.part);
if (projected.tool) tools.push(projected.tool);
// ...
if (previousTools?.length === tools.length && previousTools.every((tool, index) => tool === tools[index]))
  toolList = previousTools;
```
```tsx
// Before (ComposerSurface): handlers depend on draft, attachments, onSend, onStop
<Composer onSend={handleSend} onStop={onStop} labels={{ send: ..., stop: ... }} />

// After: stable callbacks that read the latest values from a ref
const latest = useRef({ attachments, draft, onSend, onStop });
useLayoutEffect(() => {
  latest.current = { attachments, draft, onSend, onStop };
});
const handleStop = useCallback(() => latest.current.onStop(), []);
const composerLabels = useMemo(() => ({ send: sendLabel, stop: stopLabel }), [sendLabel, stopLabel]);
```

## Takeaway
During streaming, cache derived view objects by source identity so unchanged rows keep the same reference, route changing values through refs for stable action callbacks, and debounce persisted drafts.
