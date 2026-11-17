# perf(navigation): freeze hidden tabs
- **Repository:** mrzmyr/pixy-mood-tracker-app
- **PR Link:** <https://github.com/mrzmyr/pixy-mood-tracker-app/pull/325>
- **Merged Date:** 2026-09-25
- **Category:** Rendering

## Problem
Bottom tabs stay mounted after their first visit, so a visited Statistics tab kept re-rendering its charts on every log change while hidden. With many entries this cost about 550 ms per saved entry on an iPhone 14 Pro. Measured on that device (Release build, 9,376 entries, medians) this PR alone changed add-entry to next frame from 859 ms to 300 ms, edit from 842 ms to 300 ms and delete from 850 ms to 283 ms. Opening the Statistics tab (1,132 ms to 1,134 ms) and switching back to Calendar were unchanged.

## Solution
- Set `freezeOnBlur: true` in the tab navigator screen options, which uses the react-native-screens freeze.
- Hidden tabs skip renders and catch up once when focused again; the Statistics tab still reloads on its `focus` listener.
- The PR reports passing e2e flows for saving an entry then opening Statistics, tab switching, and the Calendar filter button while the tab is hidden.

## Code Pattern (Before vs. After)
```tsx
// Before
headerStyle: {
  borderBottomColor: "#fff",
},
// (no freezeOnBlur)

// After
headerStyle: {
  borderBottomColor: "#fff",
},
// Hidden tabs skip renders until focused again. Otherwise a visited
// Statistics tab recomputes its charts on every saved entry.
freezeOnBlur: true,
```

## Takeaway
Navigators that keep screens mounted should freeze blurred screens so expensive hidden screens, such as charts bound to frequently changing data, do not re-render.
