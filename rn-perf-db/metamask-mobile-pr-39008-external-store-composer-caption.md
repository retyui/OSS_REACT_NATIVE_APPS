# perf(social): stop the post composer re-rendering on every keystroke cp-8.20.0
- **Repository:** MetaMask/metamask-mobile
- **PR Link:** <https://github.com/MetaMask/metamask-mobile/pull/39008>
- **Merged Date:** 2026-11-06
- **Category:** Rendering

## Problem
The post composer held the caption in `useState` on the screen component, so every keystroke re-rendered the whole screen: header, close button, avatar, input and the position card with its live price provider. The author notes this was verified only with a render-count test and React DevTools highlights, with no profiler numbers and no Android testing.

## Solution
- Move the caption into a small external store read through `useSyncExternalStore`; only the input, the character count and the Post button subscribe.
- `handlePost` reads the caption from the store at tap time, so it is not recreated per keystroke.
- `previewItem` no longer depends on the text (the preview never rendered it).
- The count uses `tabular-nums` so the header's end-accessory width stays stable while typing, avoiding extra header re-renders.
- Selectors must return primitives, otherwise the snapshot check fails and re-renders every keystroke.

## Code Pattern (Before vs. After)
```tsx
// Before
const [text, setText] = useState('');
const handleChangeText = useCallback((next: string) => {
  setText(next);
}, []);
// ...
<TextInput value={text} onChangeText={handleChangeText} /* ... */ />

// After
const createComposerTextStore = () => {
  let text = '';
  const listeners = new Set<() => void>();
  return {
    get: () => text,
    set: (next: string) => {
      if (next === text) return;
      text = next;
      listeners.forEach((listener) => listener());
    },
    subscribe: (listener: () => void) => {
      listeners.add(listener);
      return () => { listeners.delete(listener); };
    },
  };
};

const useComposerText = <T,>(
  store: ComposerTextStore,
  select: (text: string) => T,
): T => useSyncExternalStore(store.subscribe, () => select(store.get()));

const ComposerCharacterCount = ({ store }: { store: ComposerTextStore }) => {
  const count = useComposerText(store, (value) => value.length);
  // ...
};
```

## Takeaway
For fast-changing input state that only a few leaf components display, keep it outside the parent's React state and subscribe leaves with `useSyncExternalStore`.
