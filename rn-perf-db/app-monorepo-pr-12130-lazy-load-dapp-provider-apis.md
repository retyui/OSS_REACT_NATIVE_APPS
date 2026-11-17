# perf(kit-bg): lazy-load dapp ProviderApis + trim startup bundle (jsrsasign/lottie/lodash)
- **Repository:** OneKeyHQ/app-monorepo
- **PR Link:** <https://github.com/OneKeyHQ/app-monorepo/pull/12130>
- **Merged Date:** 2026-06-29
- **Category:** Bundle Size (+ Startup)

## Problem
`backgroundProviders.ts` registered each chain's dapp `ProviderApi` through `require()` inside lazy getters. Webpack (web) and Metro's split-bundle serializer (native) still treat a static `require()` as a build-time dependency, so every chain's provider and heavy SDK (Aptos about 5MB, Tron, NEO, Alephium, Cosmos, Sui and others) stayed in the startup graph even though Home never uses them. For native iOS the PR reports `startup.background` going from 40.77 MB to 27.50 MB (-13.27 MB, -33%, -1199 modules). On web, initial JS gzip went from 7.01 MB to 4.68 MB. Only the first commit (the provider change) was measured on native.

## Solution
- Replaced the `require()` getters with a registry of `import()` loaders (`providerApiLoaders`), so each chain's provider and SDK become a lazy chunk (webpack async chunk on web, split-bundle segment on native).
- Added `BackgroundApiBase.getProviderApi(scope)`: it returns the cached instance, or loads the module and constructs it, and de-duplicates concurrent loads through an in-flight promise cache. A rejected promise is evicted so a transient load failure is retried.
- `$private` stays eager because several callers access it synchronously by name. The one synchronous `ethereum` consumer now uses `await`, and broadcasts only iterate over providers that are already loaded.
- Other cuts in the same PR were web-only (removing the unused `jsrsasign` RSA encryptor, lazy `lottie-react` on web, scoping the webpack lodash cache group).
- Caveat from the PR: live dapp dispatch on a real device was not covered by automated checks and needs manual QA.

## Code Pattern (Before vs. After)
```ts
// Before: a static require() still anchors the module in the startup graph
Object.defineProperty(backgroundProviders, IInjectedProviderNames.ethereum, {
  get() {
    const ProviderApiEthereum = (
      require('./ProviderApiEthereum') as unknown as typeof import('./ProviderApiEthereum')
    ).default;
    const value = new ProviderApiEthereum({ backgroundApi });
    Object.defineProperty(this, IInjectedProviderNames.ethereum, { value });
    return value;
  },
  configurable: true,
  enumerable: true,
});

// After: dynamic import() creates a lazily loaded chunk
export const providerApiLoaders: Partial<
  Record<IInjectedProviderNames, () => Promise<IProviderApiModule>>
> = {
  [IInjectedProviderNames.ethereum]: () => import('./ProviderApiEthereum'),
  [IInjectedProviderNames.solana]: () => import('./ProviderApiSolana'),
  [IInjectedProviderNames.aptos]: () => import('./ProviderApiAptos'),
  // ...
};
```

```ts
// After: BackgroundApiBase
async getProviderApi(scope: IInjectedProviderNames): Promise<ProviderApiBase> {
  const existing = this.providers[scope];
  if (existing) return existing;
  let loading = this.providerApiLoadingCache[scope];
  if (!loading) {
    loading = providerApiLoaders[scope]!()
      .then((module) => {
        const instance = new module.default({ backgroundApi: this });
        this.providers[scope] = instance;
        return instance;
      })
      .catch((error) => {
        delete this.providerApiLoadingCache[scope];
        throw error;
      });
    this.providerApiLoadingCache[scope] = loading;
  }
  return loading;
}
```

## Takeaway
A `require()` inside a lazy getter does not remove a module from the startup bundle in a split-bundle setup. Use `import()` with a deduplicating loader so heavy per-feature SDKs load on first use.
