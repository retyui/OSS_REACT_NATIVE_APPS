# chore: remove barrel import of date-fns to reduce bundle size
- **Repository:** rainbow-me/rainbow
- **PR Link:** <https://github.com/rainbow-me/rainbow/pull/7166>
- **Merged Date:** 2026-02-23
- **Category:** Bundle Size

## Problem
Importing from the `date-fns/locale` barrel eagerly requires all 75 locales through CommonJS, and Metro cannot tree-shake them. The app uses only 13. The PR states the change reduces the JS bundle by 0.61 MB.

## Solution
Import each used locale from its own path (`date-fns/locale/<name>`) so only those 13 are bundled. A scoped eslint-disable for `import/no-duplicates` is added, because all subpaths resolve to the same typings file and trigger a false positive.

## Code Pattern (Before vs. After)
```ts
// Before
import { enUS, es, fr, hi, id, ja, ptBR, ru, tr, zhCN, ar, th, ko } from 'date-fns/locale';

// After
import ar from 'date-fns/locale/ar-SA';
import enUS from 'date-fns/locale/en-US';
import es from 'date-fns/locale/es';
import fr from 'date-fns/locale/fr';
import hi from 'date-fns/locale/hi';
import id from 'date-fns/locale/id';
import ja from 'date-fns/locale/ja';
import ko from 'date-fns/locale/ko';
import ptBR from 'date-fns/locale/pt-BR';
import ru from 'date-fns/locale/ru';
import th from 'date-fns/locale/th';
import tr from 'date-fns/locale/tr';
import zhCN from 'date-fns/locale/zh-CN';
```

## Takeaway
Avoid barrel imports of CommonJS libraries with many sub-modules (locales, icons) under Metro; import the specific files you need.
