# perf(markdown): reduce room-open render cost with parse cache and compiler memoization
- **Repository:** RocketChat/Rocket.Chat.ReactNative
- **PR Link:** <https://github.com/RocketChat/Rocket.Chat.ReactNative/pull/7348>
- **Merged Date:** 2026-06-02
- **Category:** Rendering

## Problem
Opening a room spent significant time re-parsing and re-rendering markdown for every message. The PR benchmark (iOS simulator, dev build, React profiler, iPhone 16e, rooms list -> DM -> scroll -> back) reports the primary Markdown mount batch (11 instances) dropping from 243.9 ms to 48.1 ms self time (517 ms to 134 ms commit), and total Markdown self time across 44 mounts from 295.8 ms to 69.8 ms. The author caveats that the test room is mostly system messages and numbers are dev-mode.

## Solution
- Add a parse cache (a `Map` capped at 200 entries, oldest evicted first) so identical message strings skip `parse()`.
- Translated messages bypass the cache and still re-parse.
- Extract block rendering into a separate `MarkdownBlockView` component with the React Compiler `'use memo'` directive, so each block only re-renders when its own `block` prop changes.
- Build the context value once per render in a compiled component so its identity is stable, and move the inline `{ gap: 2 }` style into `StyleSheet`.
- The PR also adds `key`s to the block list, which were missing before.

## Code Pattern (Before vs. After)
```tsx
// Before
tokens = !isTranslated && md ? md : parse(typeof msg === 'string' ? msg : String(msg || ''));
// ...
<View style={{ gap: 2 }}>
	<MarkdownContext.Provider value={{ mentions, channels, useRealName, username, navToRoomInfo, getCustomEmoji, onLinkPress, textStyle }}>
		{tokens?.map(block => {
			switch (block.type) {
				case 'PARAGRAPH':
					return <Paragraph value={block.value} />;
				// ...
			}
		})}
	</MarkdownContext.Provider>
</View>

// After
const PARSE_CACHE_MAX = 200;
const parseCache = new Map<string, Root>();

const parseMessage = (msg: string): Root => {
	const cached = parseCache.get(msg);
	if (cached) return cached;
	const result = parse(msg);
	if (parseCache.size >= PARSE_CACHE_MAX) {
		const oldestKey = parseCache.keys().next().value;
		if (oldestKey !== undefined) parseCache.delete(oldestKey);
	}
	parseCache.set(msg, result);
	return result;
};

const MarkdownBlockView = ({ block }: { block: MarkdownBlock }) => {
	'use memo';
	switch (block.type) {
		case 'PARAGRAPH':
			return <Paragraph value={block.value} />;
		// ...
	}
};

// in Markdown (also 'use memo')
<View style={styles.blocks}>
	<MarkdownContext.Provider value={contextValue}>
		{tokens.map((block, index) => (
			<MarkdownBlockView key={`${block.type}-${index}`} block={block} />
		))}
	</MarkdownContext.Provider>
</View>
```

## Takeaway
Cache pure, expensive parse results keyed by input string (with a bounded size), and split per-item rendering into a memoized child so list updates don't re-render unchanged blocks.
