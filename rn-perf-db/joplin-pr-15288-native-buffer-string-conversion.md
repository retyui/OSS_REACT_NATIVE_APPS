# Mobile: Resolves #15287: Refactor (native) Crypto module to improve encryption performance on mobile
- **Repository:** laurent22/joplin
- **PR Link:** <https://github.com/laurent22/joplin/pull/15288>
- **Merged Date:** 2026-05-12
- **Category:** Native (+ Storage & IO)

## Problem
The mobile E2EE crypto module converted between buffers and hex/base64/utf16le strings with the JS `Buffer` polyfill (`Buffer.from()` and `toString()`), which is slow on Hermes for large payloads during encryption and decryption. The PR states that hex, utf16le and base64 encode/decode performance is significantly improved, but does not give numbers.

## Solution
- Bumped `react-native-quick-crypto` to 1.1.2 to get the native `ab2str()` API.
- Added `Crypto.bufferToString(buffer, encoding)`. On mobile it wraps `QuickCrypto.ab2str`; on desktop it uses `Buffer.toString()` or a zero-copy `Buffer` view over the `Uint8Array`.
- On mobile, decoding strings now uses `QuickCrypto.binaryLikeToArrayBuffer()` instead of `Buffer.from()`, and ciphertext is split with `Uint8Array` views over the `ArrayBuffer` rather than `subarray` on a `Buffer`.
- `CryptoBuffer` is now `Uint8Array<ArrayBuffer>` (no `toString()`), and call sites in `EncryptionService` use `bufferToString`. A test for `bufferToString` was added to the integration tests.

## Code Pattern (Before vs. After)
```ts
// Before
const salt = Buffer.from(data.salt, 'base64');
const iv = Buffer.from(data.iv, 'base64');
// ...
const decrypted = decryptRaw(Buffer.from(data.ct, 'base64'), /* ... */);
// ...
result.iv = iv.toString('base64');
result.ct = encrypted.toString('base64');

// After
const salt = QuickCrypto.binaryLikeToArrayBuffer(data.salt, 'base64');
const iv = QuickCrypto.binaryLikeToArrayBuffer(data.iv, 'base64');
// ...
const decrypted = decryptRaw(QuickCrypto.binaryLikeToArrayBuffer(data.ct, 'base64'), /* ... */);
// ...
result.iv = crypto.bufferToString(iv, 'base64');
result.ct = crypto.bufferToString(encrypted, 'base64');

// ...
bufferToString: (buffer: CryptoBuffer, encoding: CryptoBufferEncoding) => {
	return QuickCrypto.ab2str(buffer.buffer, encoding, buffer.byteOffset, buffer.byteOffset + buffer.byteLength);
},
```

## Takeaway
Route hot-path base64/hex conversions through a native implementation instead of the JS `Buffer` polyfill, and hide the difference behind a small platform interface.
