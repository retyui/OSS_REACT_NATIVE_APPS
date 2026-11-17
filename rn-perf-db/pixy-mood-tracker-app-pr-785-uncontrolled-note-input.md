# perf(logger): keep note input uncontrolled while typing
- **Repository:** mrzmyr/pixy-mood-tracker-app
- **PR Link:** <https://github.com/mrzmyr/pixy-mood-tracker-app/pull/785>
- **Merged Date:** 2026-10-18
- **Category:** Rendering

## Problem
The logger's note input was controlled, so every keystroke called `setMessage` and a new draft state re-rendered the Logger, all slides, the header and the carousel. In a Jest test with a React `Profiler` around `Logger` and 20 keystrokes, the PR reports 20 commits before and 1 after.

## Solution
- The native `TextInput` now owns the text while typing; `SlideMessage` passes a fixed `defaultValue` (captured once with `useState`) instead of `value`.
- `typeMessage` writes only to a `latest` draft ref and updates state only when `isDirty` or `hasContent` flips (first key, or note emptied or refilled). `commit` and other setters read the ref, so a save works without blur.
- `flushMessage` publishes the ref to `draft` on blur and on unmount; slide changes dismiss the keyboard, which triggers blur.
- `TextArea` gained optional `defaultValue` and `onBlur`; controlled callers are unchanged.
- A new test asserts at most 2 Logger commits for 20 keystrokes.

## Code Pattern (Before vs. After)
```tsx
// Before
const { draft, setMessage } = useLogDraft();
// ...
<TextArea
  value={draft.message}
  onChange={setMessage}
  maxLength={MAX_MESSAGE_LENGTH}
/>

// setMessage: every keystroke sets state
setMessage: (message) => patch({ message }),

// After
const { draft, typeMessage, flushMessage } = useLogDraft();
const [initialMessage] = useState(draft.message);
useEffect(() => flushMessage, [flushMessage]);
// ...
<TextArea
  defaultValue={initialMessage}
  onChange={typeMessage}
  onBlur={flushMessage}
  maxLength={MAX_MESSAGE_LENGTH}
/>

// typeMessage: ref write; state only when a flag flips
typeMessage: (message) => {
  latest.current = { ...latest.current, message };
  const hasContent = hasDraftContent(latest.current);
  if (!state.isDirty || hasContent !== state.hasContent) {
    setState({ draft: latest.current, isDirty: true, hasContent });
  }
},
```

## Takeaway
For text inputs inside a large screen, keep the input uncontrolled and store the live text in a ref, updating React state only when derived flags change or on blur.
