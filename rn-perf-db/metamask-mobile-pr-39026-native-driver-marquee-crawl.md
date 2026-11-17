# perf(social): stop the hot token rail pinning the UI thread cp-8.20.0
- **Repository:** MetaMask/metamask-mobile
- **PR Link:** <https://github.com/MetaMask/metamask-mobile/pull/39026>
- **Merged Date:** 2026-11-06
- **Category:** Animations

## Problem
The Trending hot-token rail ran a Reanimated `useFrameCallback` for its marquee. Every frame wrote `translateX`, and with the synchronous UI-props flags off each write became a Fabric commit of the feed tree, so the UI thread stayed busy for as long as the tab was open. The PR reports no numbers.

## Solution
- Replace the per-frame shared-value write with a native-driver `Animated.timing` loop (linear easing, one timing per lap). The transform is written straight to the view and JS wakes only when a lap wraps (the loop copy hides the wrap).
- Position is predicted on the JS clock (`readOffset`) for presses and for resuming after a stop, since a native value is only readable asynchronously.
- Behavior kept: the rail stops off-screen, on low-end hardware, when chips fit, or when a chip is selected; dragging scrubs it and the crawl resumes from the release point.

## Code Pattern (Before vs. After)
```tsx
// Before
const tickMarquee = useCallback(
  (frame: { timeSincePreviousFrame: number | null }) => {
    'worklet';
    if (paused.value) return;
    const next = advanceMarquee(
      offset.value, speed.value,
      parked.value ? 0 : MARQUEE_PIXELS_PER_SECOND,
      frame.timeSincePreviousFrame ?? 0,
      trackWidthSv.value, viewportWidthSv.value,
    );
    offset.value = next.offset;
    speed.value = next.speed;
  },
  [offset, parked, paused, speed, trackWidthSv, viewportWidthSv],
);
const marqueeFrameCallback = useFrameCallback(tickMarquee, false);
const trackStyle = useAnimatedStyle(() => ({
  transform: [{ translateX: -offset.value }],
}));

// After (useRailCrawl.ts)
Animated.timing(shift, {
  toValue: width,
  duration: ((width - position) / MARQUEE_PIXELS_PER_SECOND) * 1000,
  easing: Easing.linear,
  useNativeDriver: true,
}).start(({ finished }) => {
  if (finished) {
    travel(0);
  }
});
// in the component:
const { translateX, readOffset, hold, scrubBy, release } = useRailCrawl({ /* ... */ });
const trackStyle = { transform: [{ translateX }] };
```

## Takeaway
For continuous ticker animations, a native-driver timing loop that wakes JS once per lap is cheaper than a per-frame worklet that triggers a commit each frame.
