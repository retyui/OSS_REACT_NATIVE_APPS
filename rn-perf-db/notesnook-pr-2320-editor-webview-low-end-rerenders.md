# Improved mobile editor performance on low-end devices
- **Repository:** streetwriters/notesnook
- **PR Link:** <https://github.com/streetwriters/notesnook/pull/2320>
- **Merged Date:** 2023-04-15
- **Category:** Rendering

## Problem
The mobile editor (a WebView running TipTap) was slow on low-end devices. The native side used the `useKeyboard` hook, so every keyboard show/hide re-rendered the editor events hook and re-sent the full settings object to the WebView. Inside the WebView, a 3-second `setInterval` counted words, and the header and tags components re-rendered on unrelated state changes. The PR body is empty and gives no measurements.

## Solution
- Keyboard state is no longer part of `settings`. Native listens to `Keyboard` events directly in an effect and sets a global in the WebView (`globalThis['keyboardShown']`), which the task-item component reads at tap time. This removes `isKeyboardOpen` from the toolbar store and the editor options.
- The polling word-count interval is replaced by a debounced `countWords()` (300 ms after changes, 5 s after content change) that clears the previous timer.
- `Header` is wrapped in `React.memo` with a comparator on the few props it uses; `Tags` is memoized to never re-render from props.
- In `useEditor`, redundant effects were removed and `restoreEditorState` now runs from `onReady` once the editor is loaded.

## Code Pattern (Before vs. After)
```tsx
// Before
const { keyboardShown } = useKeyboard();

useEffect(() => {
  editor.commands.setSettings({
    // ...
    keyboardShown: keyboardShown || false,
  });
}, [/* ... */, keyboardShown]);

// WebView (statusbar.tsx)
interval.current = setInterval(statusBar.current.updateWords, 3000);
```
```tsx
// After
useEffect(() => {
  const handleKeyboardDidShow: KeyboardEventListener = () => editor.commands.keyboardShown(true);
  const handleKeyboardDidHide: KeyboardEventListener = () => editor.commands.keyboardShown(false);
  const subscriptions = [
    Keyboard.addListener("keyboardDidShow", handleKeyboardDidShow),
    Keyboard.addListener("keyboardDidHide", handleKeyboardDidHide)
  ];
  return () => subscriptions.forEach((s) => s.remove());
}, [editor.commands]);

// WebView (useEditorController.ts)
function countWords(ms = 300) {
  if (typeof timers.current.wordCounter === "number") clearTimeout(timers.current.wordCounter);
  timers.current.wordCounter = setTimeout(() => statusBar?.current?.updateWords(), ms);
}

export default React.memo(Header, (prev, next) =>
  !(prev.settings.deviceMode !== next.settings.deviceMode || prev.noHeader !== next.noHeader /* ... */));
```

## Takeaway
Keep high-frequency native state (keyboard) out of props/settings sent across the WebView bridge, replace polling with debounced work, and memoize static WebView components.
