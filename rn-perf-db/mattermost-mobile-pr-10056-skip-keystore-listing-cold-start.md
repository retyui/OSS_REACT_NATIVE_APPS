# perf: skip Android Keystore listing on cold start
- **Repository:** mattermost/mattermost-mobile
- **PR Link:** <https://github.com/mattermost/mattermost-mobile/pull/10056>
- **Merged Date:** 2026-08-27
- **Category:** Startup (+ Storage & IO)

## Problem
On cold start, `initialize()` called `getAllServerCredentials()`, which on Android enumerated every generic-password Keystore service and decrypted each entry before the app database was opened. The PR reports a release APK on a Galaxy S25 Ultra (Android 16, n=8, force-stop until the Home channel rows appear) going from 16612 ms to 7246 ms median, which is 9.4 s or 56% faster.

## Solution
- Split `DatabaseManager.init` into `initAppDatabase()` and `initServerDatabases(urls)` so the app DB can be opened first.
- Query the active (logged-in) servers from the app DB and pass those URLs to `getAllServerCredentials(knownServerUrls)`, so only known servers are decrypted and no Keystore listing happens. If the URL list is empty (fresh or wiped DB), it still falls back to listing.
- Decrypt credentials in parallel with `Promise.all` instead of a sequential `for await`.
- Cache the result in memory, and keep the cache consistent in `setServerCredentials`, `removeServerCredentials` and `removePreauthSecret`; `setServerCredentials` now awaits Keychain writes and only updates the cache when they succeed.
- Trade-off stated in the code: Keystore entries with no matching active DB row are skipped.

## Code Pattern (Before vs. After)
```ts
// Before
serverCredentials = await getAllServerCredentials();
const serverUrls = serverCredentials.map((credential) => credential.serverUrl);

await DatabaseManager.init(serverUrls);
await NetworkManager.init(serverCredentials);

// getAllServerCredentials: list every Keystore service, then decrypt one by one
serverUrls = await KeyChain.getAllGenericPasswordServices();
for await (const serverUrl of serverUrls) {
    const serverCredential = await getServerCredentials(serverUrl);
    if (serverCredential) {
        serverCredentials.push(serverCredential);
    }
}

// After
await DatabaseManager.initAppDatabase();

// Keystore entries with no matching active DB row are skipped (accepted vs listing every service).
const activeUrls = (await queryAllActiveServers()?.fetch() ?? []).map((s) => s.url);
serverCredentials = await getAllServerCredentials(activeUrls);

await DatabaseManager.initServerDatabases(serverCredentials.map((c) => c.serverUrl));
await NetworkManager.init(serverCredentials);

// getAllServerCredentials: only known URLs, decrypted in parallel, cached
if (knownServerUrls?.length) {
    serverUrls = knownServerUrls;
} else {
    serverUrls = await getAllKeychainServerUrls();
}
const serverCredentials = (await Promise.all(
    serverUrls.map((serverUrl) => getServerCredentials(serverUrl)),
)).filter((credential): credential is ServerCredential => Boolean(credential));
cachedServerCredentials = serverCredentials;
```

## Takeaway
On Android, avoid enumerating the Keystore at startup; look up only the entries you already know about from your own database, and cache the decrypted result.
