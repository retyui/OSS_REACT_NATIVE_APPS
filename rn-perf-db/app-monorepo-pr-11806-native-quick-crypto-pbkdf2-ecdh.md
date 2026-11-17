# perf: optimize crypto with native quick crypto
- **Repository:** OneKeyHQ/app-monorepo
- **PR Link:** <https://github.com/OneKeyHQ/app-monorepo/pull/11806>
- **Merged Date:** 2026-05-29
- **Category:** Native

## Problem
On React Native, BIP32 secp256k1 `publicFromPrivate` still ran in pure-JS `elliptic`, which made `batchGetPublicKeys` much slower than the web-embed path. BIP39 `mnemonicToSeed` (PBKDF2, 2048 iterations) and credential decrypt / AES-GCM KDF were also repeated in batch wallet and account creation flows when caches were cold or expired. The PR reports no before/after numbers; it adds perf tracing and a Gallery benchmark page to measure the backends.

## Solution
- Added `react-native-quick-crypto` (JSI-based) and used it on native for PBKDF2: BIP39 `mnemonicToSeed` / `mnemonicToSeedSync` and the app PBKDF2 module, as a selectable backend with fallbacks for web and Jest.
- `EllipticECWrapper.publicFromPrivate` now calls a native ECDH (`createECDH`) on native for `secp256k1` and `prime256v1`, then compresses the point. Web and Jest keep the `elliptic` path.
- Cached secret material is kept as `Buffer` with sliding expiration, so active batch flows avoid re-running credential decrypt and KDF work. Explicit cache clearing was added.
- Added opt-in perf trace hooks around BIP32 and mnemonic derivation, and a flag to skip the sign/verify self-check in `N()`.
- Risk noted in the PR: high, covering output parity, native module availability, Android PBKDF2 iteration changes and cache lifetime.

## Code Pattern (Before vs. After)
```ts
// Before (curves/elliptic.ts)
publicFromPrivate(privateKey: Buffer): Buffer {
  checkBufferIsNotEmpty(privateKey);
  return Buffer.from(
    this.curve.keyFromPrivate(privateKey).getPublic().encodeCompressed(),
  );
}

// After
publicFromPrivate(privateKey: Buffer): Buffer {
  checkBufferIsNotEmpty(privateKey);

  if (this.quickCryptoCurveName && canUseQuickCryptoEC()) {
    return publicFromPrivateByQuickCrypto({
      curveName: this.quickCryptoCurveName,
      privateKey,
    });
  }
  return Buffer.from(
    this.curve.keyFromPrivate(privateKey).getPublic().encodeCompressed(),
  );
}

function publicFromPrivateByQuickCrypto({ curveName, privateKey }) {
  const ecdh = RN_QUICK_CRYPTO.createECDH(curveName);
  ecdh.setPrivateKey(privateKey);
  return compressPublicKey(Buffer.from(ecdh.getPublicKey()));
}
```

```ts
// After (bip39.ts): native PBKDF2 for the mnemonic seed
if (platformEnv.isNative) {
  return mnemonicToSeedSyncByQuickCrypto(mnemonic, passphrase, perfTrace);
}
// ...
RN_QUICK_CRYPTO.pbkdf2Sync(
  mnemonicBuffer, saltBuffer, BIP39_PBKDF2_ITERATIONS, BIP39_SEED_KEY_LENGTH, 'sha512',
);
```

## Takeaway
Move hot, CPU-bound crypto (PBKDF2, EC point math) off the JS engine to a JSI native module, keep a JS fallback for other platforms, and cache derived secrets for the duration of batch flows.
