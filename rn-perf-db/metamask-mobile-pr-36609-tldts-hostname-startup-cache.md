# perf(rpc-domains): extract hostnames with tldts instead of the URL polyfill at startup
- **Repository:** MetaMask/metamask-mobile
- **PR Link:** <https://github.com/MetaMask/metamask-mobile/pull/36609>
- **Merged Date:** 2026-09-25
- **Category:** Startup (+ Storage & IO)

## Problem
`initializeRpcProviderDomains` parsed every RPC endpoint of every chain in the cached safe-chains list (2,763 chains / 4,123 URLs) with the `whatwg-url-without-unicode` polyfill on each startup. The PR attributes about 1,000 ms of a Galaxy A14 cold start to this in a release-build CPU profile (referenced from #36044). In an M1 Node benchmark, `tldts` `getHostname` took 21.9 ms to 0.53 ms per pass (about 41x faster).

## Solution
- Use `tldts` `getHostname`, which reads the authority component directly instead of running a full WHATWG parse.
- Persist the computed hostname set via `StorageWrapper` so later cold starts read it back rather than re-parsing; a subscription on `SAFE_CHAINS_CACHE` recomputes it when the list refreshes.
- Trade-off noted in the PR: `tldts` adds about 46.5 KB min+gz of bundle (PSL trie), and percent-encoded hosts are now classified as invalid (stricter).

## Code Pattern (Before vs. After)
```ts
// Before
for (const chain of chainsList) {
  if (chain.rpc && Array.isArray(chain.rpc)) {
    for (const rpcUrl of chain.rpc) {
      try {
        const url = new URL(rpcUrl);
        newKnownDomainsSet.add(url.hostname.toLowerCase());
      } catch (e) {
        continue; // Skip invalid URLs
      }
    }
  }
}

// After
const persistedHostnames = await RPC_DOMAINS_HOSTNAMES.get();
if (persistedHostnames) {
  state.setKnownDomainsSet(new Set<string>(persistedHostnames));
} else {
  await RPC_DOMAINS_HOSTNAMES.computeAndSet();
}

// extractHostnames
const hostname = getHostname(rpcUrl);
if (hostname) {
  hostnames.add(hostname);
}
```

## Takeaway
Avoid the JS URL polyfill for bulk hostname extraction on the startup path, and persist derived data instead of recomputing it each launch.
