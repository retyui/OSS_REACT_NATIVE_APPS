# Improve cold start
- **Repository:** mattermost/mattermost-mobile
- **PR Link:** <https://github.com/mattermost/mattermost-mobile/pull/6868>
- **Merged Date:** 2022-12-15
- **Category:** Startup

## Problem
Cold start was importing heavy modules that are not needed to show the first screen. The syntax-highlight library is very large, and the `Emoji` component requires `emoji.json` (1.1 MB). On launch the announcement banner (for licensed servers) mounts `RemoveMarkdown`/`Markdown`, which import `Emoji` and `SyntaxHighlight`, and DM channel items on the home screen also use `Emoji`. The PR gives no before/after timings; it says to compare release builds against main.

## Solution
- Used inline `require()` so the modules still load, but only when a component first renders instead of at app startup (this relies on Metro inline requires).
- Split the emoji component: the real implementation moved to `emoji/emoji.tsx` and `emoji/index.tsx` became a thin wrapper that lazily requires it once and caches it in a module variable.
- Did the same for `SyntaxHighlighter` in the markdown code block.
- Extracted the props into `types/components/*.ts` so the wrappers can be typed without importing the heavy modules. `Hashtag` import was also changed to a relative path.

## Code Pattern (Before vs. After)
```tsx
// Before
import SyntaxHighlighter from '@components/syntax_highlight';
// ...
const MarkdownCodeBlock = ({language = '', content, textStyle}: MarkdownCodeBlockProps) => {
    const theme = useTheme();
    // ...

// After
import type {SyntaxHiglightProps} from '@typings/components/syntax_highlight';

let syntaxHighlighter: (props: SyntaxHiglightProps) => JSX.Element;
// ...
const MarkdownCodeBlock = ({language = '', content, textStyle}: MarkdownCodeBlockProps) => {
    const theme = useTheme();
    const SyntaxHighlighter = useMemo(() => {
        if (!syntaxHighlighter) {
            syntaxHighlighter = require('@components/syntax_highlight').default;
        }

        return syntaxHighlighter;
    }, []);
```

```tsx
// emoji/index.tsx (after): thin wrapper around the lazily required component
let emojiComponent: EmojiComponent;

const EmojiWrapper = (props: Omit<EmojiProps, 'customEmojis'>) => {
    const Emoji = useMemo(() => {
        if (!emojiComponent) {
            emojiComponent = require('./emoji').default;
        }
        return emojiComponent;
    }, []);

    return (<Emoji {...props}/>);
};
```

## Takeaway
Wrap rarely-needed heavy components (large data files, big libraries) in a lazy `require` so they stay out of the startup path.
