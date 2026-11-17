# perf(mobile): 减少长回复挂载与流式 Markdown 重复解析
- **Repository:** makecindy/cindy
- **PR Link:** <https://github.com/makecindy/cindy/pull/7726>
- **Merged Date:** 2026-11-01
- **Category:** Rendering (+ Lists)

## Problem
In the mobile companion chat, long, format-heavy replies were slow: the Markdown inline parser re-validated the same image/link/path candidates repeatedly, a streaming single paragraph re-parsed already completed content on every append, and every plain span (text, bold, inline code) mounted wrappers for share-tap context and path validation. Reported simulator numbers (iPhone simulator, Debug/Hermes, median of 3, before to after): cumulative parse time for an 18,180-character paragraph streamed in 285 appends went 7652 ms to 561 ms; longest JS frame gap on first mount of an 18,216-character formatted reply with image markers went 2199 ms to 1147 ms; React Fiber count for that case went 17,873 to 8,272. The PR notes these are parser-only or simulator measurements, not production FPS, and that very long pure-formatted replies still take about 2.4 s to mount.

## Solution
- The inline regex search now caches validated candidate results (`search.next(key, from, build)`), reusable only while the cursor stays inside the earliest raw match range.
- For paragraphs of 1024+ characters, the parser keeps a per-block `WeakMap` snapshot of the stable inline prefix. When the new text starts with the previous source, parsing resumes from the last safe offset instead of from 0. The prefix is committed only through whitespace after a narrow whitelist of complete atoms (words, `**bold**`, simple inline code); unclosed or complex syntax stays in the re-parsed tail.
- Non-append edits fall back to a full parse, and old parse results stay immutable; snapshots are freed with their blocks.
- In the renderer, plain text, bold, emphasis, strike, math and non-path inline code render as a raw `UITextView` child (or bare string) instead of the share-selection context wrapper; only links, images and real path candidates keep the interactive components.
- Correctness was checked with a seeded differential test of old full parser vs. new incremental parser (28,147 cases) with identical output.

## Code Pattern (Before vs. After)
```ts
// Before
const candidates = [
  matchMarkdownImage(input, from, context),
  matchHtmlImage(input, from, context),
  // ...
];

pushBlock({
  type: 'paragraph',
  key: `p:${lineNumber(lineIndex)}:${blockNumber()}`,
  inlines: parseMobileMarkdownInlines(text, paragraphStartsInComment),
}, paragraphStart, lineIndex);

// After
const candidates = [
  context.search.next('MarkdownImage', from, () => matchMarkdownImage(input, from, context)),
  context.search.next('HtmlImage', from, () => matchHtmlImage(input, from, context)),
  // ...
];

if (!paragraphStartsInComment && text.length >= 1024) {
  const previous = previousParagraphs.get(lineNumber(paragraphStart));
  const retained = previous && text.startsWith(previous.source) ? previous : undefined;
  const from = retained?.safeOffset ?? 0;
  const safeOffset = safeParagraphInlineEnd(text, from);
  const stable = joinParagraphInlines(retained?.stable ?? [],
    safeOffset > from ? parseMobileMarkdownInlines(text.slice(from, safeOffset)) : []);
  inlines = joinParagraphInlines(stable,
    safeOffset < text.length ? parseMobileMarkdownInlines(text.slice(safeOffset)) : []);
  snapshot = { source: text, safeOffset, stable };
}
```

```tsx
// Before
case 'strong':
  return <SpanText key={spanKey(`strong:${index}`)} style={[ctx.baseStyle, styles.markdownStrong]}>{inline.text}</SpanText>;
case 'code':
  return <InlineCodePathSpan /* ... */ />;

// After
const StaticSpan = ctx.SpanText === MarkdownSelectableSpan ? UITextView : (ctx.SpanText ?? Text);
// ...
case 'strong':
  return <StaticSpan maxFontSizeMultiplier={MAX_FONT_SIZE_MULTIPLIER} key={spanKey(`strong:${index}`)} style={[ctx.baseStyle, styles.markdownStrong]}>{inline.text}</StaticSpan>;
case 'code':
  if (!classifyInlineCodePathCandidate(inline.text)) {
    return <StaticSpan maxFontSizeMultiplier={MAX_FONT_SIZE_MULTIPLIER} key={spanKey(`code:${index}`)} style={[ctx.baseStyle, styles.markdownInlineCode]}>{inline.text}</StaticSpan>;
  }
```

## Takeaway
For streamed text, cache the completed prefix of the parse and re-parse only the unfinished tail, and keep passive text nodes free of gesture or context wrappers.
