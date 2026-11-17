# perf(chat-messages): hold off-screen streaming Markdown until the list end is in view
- **Repository:** CherryHQ/cherry-studio-app
- **PR Link:** <https://github.com/CherryHQ/cherry-studio-app/pull/1087>
- **Merged Date:** 2026-09-24
- **Category:** Rendering (+ Lists)

## Problem
While a long reasoning or answer part streams, every published update (every 100 ms) re-parses and re-lays out the whole part in the native Markdown renderer and re-measures it on the JS thread, whatever is on screen. Reading the top of an expanded 60K-character reasoning block while its tail grew off screen made scrolling janky. Measured on a Xiaomi device (Android, dev client, 800 chars/s stream, 25 s of scrolling), two runs each, before to after: frame time p50 32/40 ms to 15/15 ms; p90 93/129 ms to 18/18 ms; p99 150/150 ms to 26/24 ms; seconds with a >100 ms JS stall (of 25) 14/21 to 2/0. Static scrolling with no stream is 12/15/22 ms.

## Solution
- `MessageList` derives whether the growing end is near the viewport from scroll offset, content height and viewport height, using a Reanimated `useAnimatedReaction` and a worklet. It uses hysteresis (reveal within 160 pt of the bottom inset, hide beyond 320 pt) so growth at the edge cannot flap. Following mode always counts as visible.
- The boolean is published through a small context (`MessageListLiveTailProvider`).
- `PartMarkdown`, the single Markdown entry point, keeps showing the last text while the part is streaming and the end is off screen, then catches up in one render when the end comes back into view, the list follows the bottom, or the part finishes.
- Tradeoffs from the PR: the streaming message stops growing while the reader is away from the end, and following the bottom still re-lays out the full part on each update. iOS was not measured.

## Code Pattern (Before vs. After)
```tsx
// Before
export function PartMarkdown({ isStreaming, markdown }: PartMarkdownProps) {
  return <MarkdownText isStreaming={isStreaming} markdown={markdown} />;
}

// After
export function PartMarkdown({ isStreaming, markdown }: PartMarkdownProps) {
  const isLiveTailVisible = useMessageListLiveTailVisible();
  const [shownMarkdown, setShownMarkdown] = useState(markdown);
  const isHeld = isStreaming && !isLiveTailVisible;
  if (!isHeld && shownMarkdown !== markdown) {
    setShownMarkdown(markdown);
  }

  return <MarkdownText isStreaming={isStreaming} markdown={isHeld ? shownMarkdown : markdown} />;
}

// messageListLayout.ts
const LIVE_TAIL_REVEAL_MARGIN = 160;
const LIVE_TAIL_HIDE_MARGIN = 320;

export function isMessageListLiveTailInView(
  scrollOffset: number,
  contentHeight: number,
  viewportHeight: number,
  bottomInset: number,
  wasInView: boolean,
): boolean {
  'worklet';
  const distanceFromEnd = contentHeight - viewportHeight - scrollOffset;
  const margin = wasInView ? LIVE_TAIL_HIDE_MARGIN : LIVE_TAIL_REVEAL_MARGIN;
  return distanceFromEnd <= bottomInset + margin;
}
```

## Takeaway
For streamed content inside a list, stop feeding updates to the renderer for rows whose growing part is off screen, and use hysteresis on the visibility signal so it does not flicker.
