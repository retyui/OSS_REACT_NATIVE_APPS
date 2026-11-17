# fix: optimize MatrixBackground to reduce Hermes GC pressure on iOS
- **Repository:** OneKeyHQ/app-monorepo
- **PR Link:** <https://github.com/OneKeyHQ/app-monorepo/pull/10083>
- **Merged Date:** 2026-02-10
- **Category:** Memory (+ Rendering)

## Problem
A Sentry crash report showed a `SIGABRT` caused by a Hermes out-of-memory during `FinalizeWalletSetup` on iOS (an iPhone 14 with only 221MB free). The decorative `MatrixBackground` animation generated 60 new random strings every 200ms and re-rendered 60 `SizableText` nodes with them, which created continuous GC pressure during wallet creation. The PR gives no before/after numbers.

## Solution
- Split the component into `MatrixBackground.native.tsx` and `MatrixBackground.tsx`, sharing a `useMatrixPool` hook that builds the random strings once and keeps them in a `useRef`.
- Native version: renders a single `SizableText` and cycles through 10 pre-generated blocks of 3000 characters, so each tick only picks an existing string with no new allocation. The tick interval goes from 200ms to 500ms.
- Web version: keeps the multi-line layout, but picks lines at random from a pool of 120 pre-built lines.
- Also replaced the deprecated `runOnJS` with `scheduleOnRN` from `react-native-worklets`.

## Code Pattern (Before vs. After)
```tsx
// Before
const generateLines = () => {
  const newLines: string[] = [];
  for (let i = 0; i < lineCount; i += 1) {
    let line = '';
    for (let j = 0; j < charsPerLine; j += 1) {
      line += characterSet[Math.floor(Math.random() * characterSet.length)];
    }
    newLines.push(line);
  }
  setLines(() => newLines);
};
const interval = setInterval(generateLines, updateInterval);
// ... lines.map((line, idx) => <SizableText key={idx}>{line}</SizableText>)

// After (MatrixBackground.native.tsx)
const pool = useMatrixPool(TEXT_POOL_COUNT, TEXT_TOTAL_CHARS);
const [text, setText] = useState(pool[0]);

useEffect(() => {
  let index = 0;
  const tick = () => {
    index = (index + 1) % pool.length;
    setText(pool[index]);
  };
  const interval = setInterval(tick, updateInterval);
  return () => clearInterval(interval);
}, [pool, updateInterval]);

return <SizableText textAlign="center" fontFamily="$monoRegular" letterSpacing={2}>{text}</SizableText>;
```

## Takeaway
For purely decorative, timer-driven visuals, precompute the content once and rotate through it instead of allocating fresh strings and React nodes on every tick.
