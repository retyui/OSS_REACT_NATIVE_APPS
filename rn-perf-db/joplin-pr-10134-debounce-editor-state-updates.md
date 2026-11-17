# Mobile: Fixes #10130: Improve note editor performance when quickly entering text
- **Repository:** laurent22/joplin
- **PR Link:** <https://github.com/laurent22/joplin/pull/10134>
- **Merged Date:** 2024-03-20
- **Category:** Rendering

## Problem
In the mobile note editor, every editor change event immediately updated `state.note.body` in `Note.tsx`, triggering a full re-render of the screen. When typing quickly (e.g. with an external keyboard on a tablet), a render took longer than the interval between events, so events queued up. The PR reports state being set several seconds after the text was typed, and in the "before" recording the overflow menu took about 20s to appear after being tapped; the PR's test plan expects < 1s after the fix.

## Solution
- The Markdown editor's content is uncontrolled (it only receives `initialText`), so the parent does not need to re-render per keystroke.
- The `onChange` handler is now debounced by 100 ms, so `shared.noteComponent_change` (which sets state) and `scheduleSave` run at most once per burst of typing.
- The plain `TextInput` path keeps its immediate handler, since it is a controlled input.
- Side benefit noted in the PR: the last edit is less likely to be lost, because there is no long backlog of pending update events when the user leaves the screen.

## Code Pattern (Before vs. After)
```tsx
// Before
private onBodyChange(event: EditorChangeEvent) {
	shared.noteComponent_change(this, 'body', event.value);
	this.scheduleSave();
}
// ...
<NoteEditor
	initialText={note.body}
	onChange={this.onBodyChange}
	// ...
/>

// After
// Avoid saving immediately -- the NoteEditor's content isn't controlled by its props
// and updating this.state.note immediately causes slow rerenders.
private onMarkdownEditorTextChange = debounce((event: EditorChangeEvent) => {
	shared.noteComponent_change(this, 'body', event.value);
	this.scheduleSave();
}, 100);
// ...
<NoteEditor
	initialText={note.body}
	onChange={this.onMarkdownEditorTextChange}
	// ...
/>
```

## Takeaway
If a child input owns its own text, debounce the callback that mirrors that text into parent state so a heavy parent does not re-render on every keystroke.
