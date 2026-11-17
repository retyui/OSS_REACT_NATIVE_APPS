# perf(mobile): stop the waking-session freeze, cut re-renders, render markdown images
- **Repository:** kortix-ai/CompanyOS
- **PR Link:** <https://github.com/kortix-ai/CompanyOS/pull/8468>
- **Merged Date:** 2026-10-02
- **Category:** Rendering (+ Animations)

## Problem
On the mobile app the URL autolinker's bare-domain regex backtracked catastrophically under Hermes: one ordinary 3.8k-character agent reply took 19.8 s and then threw `RangeError: Maximum regex stack depth reached`. The saved-copy view renders all text parts at once, so the app froze and menu/drawer taps did nothing (V8 on web never showed it). The PR reports the same 21 fixture parts going from over 20 s plus a crash to 39 ms. Other markdown regexes took 0.4-5.8 s on 50k-char adversarial input before and 3-20 ms after.

## Solution
- Rewrite the domain label as `[a-zA-Z0-9][a-zA-Z0-9-]{0,62}\.` so each label has only one parse (no loop/last-character split inside an optional group inside `{1,64}`); reject labels ending in `-` after matching with a separate check.
- Replace the markdown-link and angle-link regexes with linear index scans that cache the next `]`, `)` or `>` position.
- Same PR also: memoize project home content, drive the "Thinking" dot-matrix glyph from a UI-thread `useFrameCallback` over a precomputed sample table (no per-frame JS work), poll the drawer list only while open, and stop a refetch-on-open timer from re-arming every render (it looped about every 0.5 s).

## Code Pattern (Before vs. After)
```ts
// Before
(?:[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?\.){1,64}(?:com|org|net|...)

const linkRe = /\[([^\]]{0,4096})\]\(([^)]{0,8192})\)/g;
while ((m = linkRe.exec(text)) !== null) {
  ranges.push([m.index, m.index + m[0].length - 1]);
}

// After
(?:[a-zA-Z0-9][a-zA-Z0-9-]{0,62}\.){1,64}(?:com|org|net|...)
const LABEL_ENDS_IN_HYPHEN = /-\./; // checked after the match

function nextIndexer(text: string, needle: string): (from: number) => number {
  let cached = -2;
  return (from) => {
    if (cached === -1 || cached >= from) return cached;
    cached = text.indexOf(needle, from);
    return cached;
  };
}
pushMarkdownLinkRanges(text, ranges);
```

```tsx
// Before: JS-thread rAF loop writing every frame
let handle = requestAnimationFrame(function tick(now) {
  frame.value = toOpacities(entry.frame(now - start, false));
  handle = requestAnimationFrame(tick);
});

// After: UI-thread frame callback only advances an index
const loop = useFrameCallback((info) => {
  'worklet';
  clock.value += info.timeSincePreviousFrame ?? 0;
  const next = trackSampleIndex(track.value.periodMs, clock.value);
  if (next !== sample.value) sample.value = next;
}, !still);
```

## Takeaway
Hermes' regex engine can backtrack far worse than V8, so avoid nested optional/overlapping quantifiers on user text and prefer index scans; test with long adversarial input on-device.
