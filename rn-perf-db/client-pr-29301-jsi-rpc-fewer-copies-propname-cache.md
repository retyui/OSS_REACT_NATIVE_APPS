# perf(react-native-kb): cut copies and interning in JSI RPC path
- **Repository:** keybase/client
- **PR Link:** <https://github.com/keybase/client/pull/29301>
- **Merged Date:** 2026-06-10
- **Category:** Native (+ Networking)

## Problem
Every RPC message between JS and the Go service crosses a C++ JSI bridge (msgpack to JSI values and back). The hot path did avoidable work per message: it re-interned the same map key strings as `PropNameID`s each time, zero-filled a new `Uint8Array` before overwriting it with `memcpy`, copied the packed payload into a second buffer to prepend the 5-byte frame header, and ran a prototype-chain `instanceOf` check on every plain object when probing for typed arrays. The PR reports no timings.

## Solution
- Cache `PropNameID`s for msgpack map keys in an `unordered_map` (capped at 4096 entries and cleared on overflow; dropped on runtime teardown/reset).
- Build binary payloads via a `MutableBuffer` subclass that copies once into uninitialized memory, avoiding the zero-fill pass.
- Reserve the 5-byte frame header at the front of the msgpack send buffer and patch it in place, removing the second `combinedBuf_` copy.
- Check `byteLength` (one property get) before `instanceOf` when probing typed arrays, so plain objects bail early.

## Code Pattern (Before vs. After)
```cpp
// Before
Value uint8Array = uint8ArrayCtor(runtime).callAsConstructor(
    runtime, static_cast<double>(size));          // zero-filled
Object uint8ArrayObj = uint8Array.asObject(runtime);
ArrayBuffer buffer = uint8ArrayObj.getProperty(runtime, "buffer")
                         .asObject(runtime).getArrayBuffer(runtime);
if (size > 0) std::memcpy(buffer.data(runtime), ptr, size);
return uint8Array;
// ...
obj.setProperty(runtime,
    jsi::PropNameID::forUtf8(runtime, reinterpret_cast<const uint8_t*>(k.via.str.ptr), k.via.str.size),
    val);

// After
ArrayBuffer buffer(runtime, std::make_shared<CopiedBuffer>(ptr, size)); // single copy
return uint8ArrayCtor(runtime).callAsConstructor(runtime, std::move(buffer));
// ...
obj.setProperty(
    runtime, mapKeyPropName(runtime, k.via.str.ptr, k.via.str.size), val); // cached PropNameID
```

```cpp
// Before: second buffer to prepend header
combinedBuf_.resize(headerLen + mp_->sendBuf.size());
std::memcpy(combinedBuf_.data(), frameHeader, headerLen);
std::memcpy(combinedBuf_.data() + headerLen, mp_->sendBuf.data(), mp_->sendBuf.size());
writeToGo_(combinedBuf_.data(), combinedBuf_.size());

// After: reserve header, patch in place
const char placeholder[headerLen] = {0};
mp_->sendBuf.write(placeholder, headerLen);
// ... pack payload ...
header[0] = 0xce; /* ... 4 big-endian length bytes ... */
writeToGo_(mp_->sendBuf.data(), mp_->sendBuf.size());
```

## Takeaway
In JSI hot paths, cache `PropNameID`s for repeated keys and avoid zero-fill plus extra copies of large buffers.
