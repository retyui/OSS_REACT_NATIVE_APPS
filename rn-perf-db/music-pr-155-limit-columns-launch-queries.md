# perf: Improve launch performance for users with lots of tracks. (1/n)
- **Repository:** MissingCore/Music
- **PR Link:** <https://github.com/MissingCore/Music/pull/155>
- **Merged Date:** 2025-01-31
- **Category:** Startup (+ Storage & IO)

## Problem
Users with large libraries reported slow app launches. With a simulated library of 1481 tracks, hydrating the Zustand "Recent List" store took ~1.85s (~2.76s if "All Tracks" was the last played list), and the onboarding/loading checks (new/modified track detection, cleanup, artwork checks) took ~1.96s. Much of it came from SQL queries that loaded every column of every track (plus album relations) when only a few fields were needed.

## Solution
- Reworked the Drizzle query helpers (`getAlbum(s)`, `getArtist(s)`, `getPlaylist(s)`, `getSpecialPlaylist`) so callers can pass `columns`, `trackColumns`, `albumColumns`, `withAlbum` and `withTracks` to fetch only the fields they need.
- The Recent List store subscription and the launch-time cleanup/artwork logic now request only the minimal columns (e.g. `["id"]` for tracks).
- Limited the Recent List to the last 15 played lists.
- Reported results (1481 tracks): store hydration ~1.85s to ~0.45s, ~2.76s to ~1.3s with "All Tracks" in the playback store, and loading-screen checks ~1.96s to ~1.3s.
- Caveat: `getPlaylist(s)` does not support skipping the tracks relation; single-item getters now always throw when nothing is found.

## Code Pattern (Before vs. After)
```ts
// Before
const data = await getAlbum(id);
entry = formatForMediaCard({ type: "album", data, t: i18next.t });
// ...
const data = await getArtist(id);

// After
const data = await getAlbum(id, {
  columns: ["id", "name", "artistName", "artwork"],
  trackColumns: ["id"],
});
// ...
const data = await getArtist(id, {
  columns: ["name", "artwork"],
  trackColumns: ["id"],
  withAlbum: false,
});
```

```ts
// Before
const oldValue = await getArtist(id);

// After
const oldValue = await getArtist(id, {
  columns: ["artwork"],
  withTracks: false,
});
```

## Takeaway
On the startup path, select only the columns and relations a consumer renders; loading full rows for thousands of records dominates launch time.
