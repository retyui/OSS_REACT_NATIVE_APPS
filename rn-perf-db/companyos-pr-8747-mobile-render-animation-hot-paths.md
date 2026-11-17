# perf(mobile): remove render and animation hot paths before store release
- **Repository:** kortix-ai/CompanyOS
- **PR Link:** <https://github.com/kortix-ai/CompanyOS/pull/8747>
- **Merged Date:** 2026-10-02
- **Category:** Rendering (+ Lists, Animations, Startup)

## Problem
A pre-release audit of the `apps/mobile` React Native app found many causes of jank. Examples from the PR: the Skia shader hero (with tilt sensor) stayed mounted behind every thread at opacity 0; transcript rows re-rendered on every 64 ms streaming tick; `react-native-markdown-display` generated new global-counter keys on each parse, so the growing last block remounted about 15 times per second; a keystroke on project home re-rendered the Skia hero; a drawer toggle re-rendered the open thread. The PR gives no timing numbers; its evidence is tests (render counts, key stability, incremental-parse equivalence) and a successful iOS/Android export, with device checks left unchecked in the test plan. Only mobile-app changes are described here; the PR is mobile-only.

## Solution
- Streaming markdown: a patch-package patch for `react-native-markdown-display@7.0.2` assigns position-based keys (parent key, index, type), so a re-parse updates views in place. Autolinking and block splitting process only newly arrived text using a bounded LRU `PrefixCache`, with tests showing output equals a full pass.
- Isolate hot state: the project-home draft moved into a child `HomeComposer`, `ProjectHero` is `React.memo`'d with no props, model resolution is `useMemo`'d, and the thread's handlers and `interruptedIds` keep stable identities.
- Unmount the hero after its fade-out; queue polling slows from 3 s to 15 s on idle threads with an empty queue.
- Navigation: memoized route context, thread view, drawer rows and composer sheets; `freezeOnBlur` on the root and project stacks; sub-session trees capped at 5 rows.
- Animation and layout: Android `TextShimmer` keeps the masked sweep, stale tools render without Lottie, `PinnedBar` follows the sheet edge with same-frame `useDerivedValue`s, and non-critical resume work is delayed 400-800 ms.
- Build: removed 11 unused dependencies and `expensify-common` from the bundle by importing `MarkdownTextInput` from its own module; `eas-build-post-install: patch-package` so EAS builds apply the patches.
- Visible trade-off from the PR: a prompt queued on another device may take up to 15 s to appear on an idle thread.

## Code Pattern (Before vs. After)
```tsx
// Before: draft state lives in ProjectHome, so every keystroke re-renders the hero
const [draft, setDraft] = React.useState(initialDraft.text);
useComposerDraft(draftKey({ kind: 'project', projectId }), draft, setDraft);
// ...
const submitNow = React.useCallback(async () => {
  const text = draft.trim();
  // ...
}, [draft, files, /* ... */]);

export function ProjectHero() {
  // Skia canvas + tilt sensor
}

// After: draft lives in a child; hero is memoized and prop-less
// The draft lives in `HomeComposer`, not here: a keystroke re-renders the
// composer card only, never the hero (a Skia canvas), the sheets or the model
// resolution.
const submitNow = React.useCallback(async (draft: string) => {
  const text = draft.trim();
  // ...
}, [files, /* ... */]);

export const ProjectHero = React.memo(function ProjectHero() {
  // Skia canvas + tilt sensor
});
```

```js
// Patch to react-native-markdown-display: stable, path-based keys
// Before
key: getUniqueID() + '_' + type,

// After
function assignKeys(nodes, prefix) {
  for (let i = 0; i < nodes.length; i++) {
    const node = nodes[i];
    node.key = prefix + i + '_' + node.type;
    assignKeys(node.children, node.key + '-');
  }
}
```

## Takeaway
For streamed content, give rendered nodes stable keys so growth updates in place, and keep fast-changing input state in a leaf component so expensive siblings such as shader canvases do not re-render.
