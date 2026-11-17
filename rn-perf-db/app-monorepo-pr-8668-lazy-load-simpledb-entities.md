# feat: speed up startup by lazy-loading DbEntity
- **Repository:** OneKeyHQ/app-monorepo
- **PR Link:** <https://github.com/OneKeyHQ/app-monorepo/pull/8668>
- **Merged Date:** 2025-10-14
- **Category:** Startup (+ Memory)

## Problem
`SimpleDb` imported about 55 entity classes at the top of the file and instantiated all of them as class fields. Every one of those modules was therefore evaluated and constructed during background-service start, whether or not the entity was needed. The PR reports no measurements; its summary only says that startup is faster and memory use is lower.

## Solution
- Replaced the eager imports and field initializers with lazy getters, one per entity.
- Each getter `require()`s its entity module on first access, builds the instance, then calls `Object.defineProperty(this, name, { value })` to replace the getter with a plain data property.
- Existing call sites such as `simpleDb.prime` keep working unchanged, and later reads cost nothing extra.
- Caveat: module-evaluation errors now surface at first access rather than at launch.

## Code Pattern (Before vs. After)
```ts
// Before
import { SimpleDbEntityPrime } from '../entity/SimpleDbEntityPrime';
import { SimpleDbEntityBrowserTabs } from '../entity/SimpleDbEntityBrowserTabs';
// ... ~55 imports

export class SimpleDb {
  prime = new SimpleDbEntityPrime();

  browserTabs = new SimpleDbEntityBrowserTabs();
  // ...
}

// After
export class SimpleDb {
  // Lazy load entities using getters
  get prime() {
    const SimpleDbEntityPrime = (
      require('../entity/SimpleDbEntityPrime') as unknown as typeof import('../entity/SimpleDbEntityPrime')
    ).SimpleDbEntityPrime;
    const value = new SimpleDbEntityPrime();
    Object.defineProperty(this, 'prime', { value });
    return value;
  }
  // ...
}
```

## Takeaway
Replace barrel-style eager instantiation of many services with memoizing lazy getters so startup only pays for what the first screen uses.
