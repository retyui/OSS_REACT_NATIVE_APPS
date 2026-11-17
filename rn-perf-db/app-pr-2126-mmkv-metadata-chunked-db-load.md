# feat: improve initial load performance on mobile
- **Repository:** standardnotes/app
- **PR Link:** <https://github.com/standardnotes/app/pull/2126>
- **Merged Date:** 2023-01-03
- **Category:** Startup (+ Storage & IO)

## Problem
On mobile, launch started by calling `getDatabasePayloads()`, which listed every AsyncStorage key and fetched every full item payload before the app could proceed to load. Sorting by priority (items keys, then priority content types, then most recently updated) needed the full payloads, so initial load cost grew with the size of the user's data. The PR description is empty and reports no numbers.

## Solution
- A new `getDatabaseLoadChunks(options, identifier)` device API returns only the keys, already sorted and split into chunks: items keys first, a content-type-priority chunk, then remaining keys in `batchSize` chunks. The sorter (`GetSortedPayloadsByPriority`) moved into `services` and works on lightweight metadata (`uuid`, `content_type`, `updated_at`).
- On mobile, that metadata is kept in an MMKV store (`react-native-mmkv`, synchronous) via `DatabaseMetadata`, so sorting needs no AsyncStorage read. If metadata is empty (first run after upgrade), it is built once from all entries.
- `SyncService.loadDatabasePayloads()` now pulls chunks itself, so the app can start before all items are loaded. Load errors surface as a `LocalDatabaseReadError` event.
- Risk: items are written to both AsyncStorage and MMKV metadata, so the two must stay in sync.

## Code Pattern (Before vs. After)
```ts
// Before (MobileDevice)
getDatabaseKeys(): Promise<string[]> {
  return AsyncStorage.getAllKeys()
}

// Application.ts
const databasePayloads = await this.syncService.getDatabasePayloads()
const loadPromise = this.syncService.loadDatabasePayloads(databasePayloads).then(/* ... */)
```
```ts
// After (mobile Database.ts)
async getLoadChunks(options: DatabaseLoadOptions): Promise<DatabaseKeysLoadChunkResponse> {
  let metadataItems = this.metadataStore.getAllMetadataItems() // MMKV, sync

  if (metadataItems.length === 0) {
    const allEntries = await this.getAllEntries()
    metadataItems = this.metadataStore.runMigration(allEntries)
  }

  const sorted = GetSortedPayloadsByPriority(metadataItems, options)
  // ...
  const remainingKeysChunks: DatabaseKeysLoadChunk[] = []
  for (let i = 0; i < remainingKeys.length; i += options.batchSize) {
    remainingKeysChunks.push({ keys: remainingKeys.slice(i, i + options.batchSize) })
  }
  return { keys: { itemsKeys: itemsKeysChunk, remainingChunks: [contentTypePriorityChunk, ...remainingKeysChunks] }, /* ... */ }
}

// Application.ts
const loadPromise = this.syncService.loadDatabasePayloads().then(/* ... */)
```

## Takeaway
Keep a small, synchronously readable index (MMKV) of what you need to order data, and load full records in prioritized chunks after the app is interactive.
