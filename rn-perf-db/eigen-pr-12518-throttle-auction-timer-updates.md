# chore: [HomeView Performance] do not re-render artwork grid items each second
- **Repository:** artsy/eigen
- **PR Link:** <https://github.com/artsy/eigen/pull/12518>
- **Merged Date:** 2025-07-23
- **Category:** Rendering

## Problem
While profiling HomeView scrolling, the author found `ArtworkAuctionTimer` updating its state every second through `setInterval`. With around 20 lots on screen, that is 20 components re-rendering each second, which can degrade scrolling, especially when a lots rail appears and many JS calls are already running. No before/after numbers are reported; QA was manual on iOS and Android.

## Solution
- Split the timer into an `AuctionTimerBid` child so the interval only exists when a bid timer is actually shown (not for "Register by" labels).
- If the lot closes in more than an hour (and bidding is not extended), do not start an interval at all.
- Otherwise tick every second when under 5 minutes remain, and every 60 seconds before that.
- Apply a similar change to `ExpiresInTimer` in the Activity feed. As merged, its guard checks that `diffNow()` is truthy (always true for a `Duration`), so it never starts an interval; that is a bug in the diff, not intended behavior.

## Code Pattern (Before vs. After)
```tsx
// Before
const INTERVAL = 1000
// ...
useEffect(() => {
  intervalId.current = setInterval(() => {
    const { time: timerTime } = getTimer(lotClosesAt ?? "")
    setTime(timerTime)
  }, INTERVAL)

  return () => {
    if (intervalId.current) clearInterval(intervalId.current)
  }
}, [])

// After
useEffect(() => {
  // Do not update the timer if the lot is closing in longer than an hour
  if (lotEndAt.diffNow().as("hours") > 1 && !onlineBiddingExtended) {
    return
  }

  // Every second if closing in less than 5 minutes, otherwise every minute
  const interval = lotEndAt.diffNow().as("minutes") < 5 ? 1000 : 60000

  intervalId.current = setInterval(() => {
    const { time: timerTime } = getTimer(lotClosesAt ?? "")
    setTime(timerTime)
  }, interval)

  return () => {
    if (intervalId.current) clearInterval(intervalId.current)
  }
}, [lotClosesAt])
```

## Takeaway
Match the update rate of countdown timers to the granularity the UI displays; a per-second state update across many list items is constant wasted render work.
