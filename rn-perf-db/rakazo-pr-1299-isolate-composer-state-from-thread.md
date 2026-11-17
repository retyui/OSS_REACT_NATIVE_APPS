# perf(mobile): keep composer typing from re-rendering the thread
- **Repository:** elie222/rakazo
- **PR Link:** <https://github.com/elie222/rakazo/pull/1299>
- **Merged Date:** 2026-10-10
- **Category:** Rendering

## Problem
The draft text, mention query and slash query state lived in the `Thread` screen component, so every keystroke re-rendered the whole thread. In a React harness, typing 20 characters caused 20 extra `Thread` renders (now 0). The PR also cites a simulator run (minified JS, 150-message thread, 40 typed characters) for the stacked PR #1295 alone: per-keystroke change-to-next-JS-frame p95 went from 89.9 ms to 12.8 ms; this PR removes the remaining parent render per keystroke.

## Solution
- Moved draft, mention/slash query, selected mentions and selected skill state, plus their handlers, into a new `ThreadComposer` component.
- `Thread` reads the latest composition via a ref handle (`composerRef.current?.snapshot()`) when sending and when delivery settles, instead of holding it in state.
- The composer is keyed by `threadKey` so it resets on thread switch; reply and attachment state stay on the screen and are passed as props.

## Code Pattern (Before vs. After)
```tsx
// Before (Thread)
const [draft, setDraft] = useState("");
const [mentionQuery, setMentionQuery] = useState<string | null>(null);
const [slashQuery, setSlashQuery] = useState<string | null>(null);
const [selectedMentions, setSelectedMentions] = useState<ComposerMention[]>([]);
const composerSnapshot: ComposerSnapshot = {
  promptText: serializeComposerPrompt(draft, selectedSkill, selectedMentions),
  // ...
};
const composerRef = useRef(composerSnapshot);
composerRef.current = composerSnapshot;

// After (Thread)
const composerRef = useRef<ThreadComposerHandle>(null);
// in send():
const submitted = composerRef.current?.snapshot();
if (!submitted) return;
// ...
<ThreadComposer
  key={threadKey}
  ref={composerRef}
  agentSkills={agentSkills}
  composerMentionTargets={composerMentionTargets}
  replyTargetId={replyTarget?.id}
  /* ... */
/>

// After (ThreadComposer)
const [draft, setDraft] = useState("");
const [mentionQuery, setMentionQuery] = useState<string | null>(null);
useImperativeHandle(ref, () => ({
  snapshot: () => ({
    promptText: serializeComposerPrompt(draft, selectedSkill, selectedMentions),
    mentions: selectedMentions,
    skill: selectedSkill,
    replyTargetId,
    replyQuote,
    attachmentIds,
  }),
  reset: () => { /* ... */ },
}));
```

## Takeaway
Keep fast-changing input state in the leaf component that owns the TextInput and expose it to the parent through an imperative handle, so typing does not re-render the screen.
