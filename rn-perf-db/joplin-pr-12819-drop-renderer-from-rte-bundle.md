# Mobile: Performance: Improve Rich Text Editor startup performance
- **Repository:** laurent22/joplin
- **PR Link:** <https://github.com/laurent22/joplin/pull/12819>
- **Merged Date:** 2025-07-30
- **Category:** Bundle Size (+ Startup)

## Problem
The Rich Text Editor runs in a WebView from a bundled, base64-processed JavaScript file. Because the bundle imported `@joplin/lib/HtmlToMd`, it also pulled in `@joplin/renderer` (> 1 MB) and the Joplin localization data (> 4 MB) via an indirect import. The PR reports the processed bundle in `app-mobile/lib/rnInjectedJs` at roughly 11 MB before the change and about 600 KB after, which should speed up editor startup.

## Solution
- Added `convertHtmlToMarkdown.ts` in the editor's content script that uses `@joplin/turndown` and the GFM plugin directly, with the same options `HtmlToMd` used (ATX headings, fenced code, `preserveColorStyles`, etc.).
- Replaced `HtmlToMd` usage in the content script with this function, avoiding the transitive imports of the renderer and localization data.
- Moved the `MarkupLanguage` enum into `@joplin/renderer/types` so the content script can import it without loading the full renderer; other files were updated to import from there.
- Moved `contentScript.ts` to `contentScript/index.ts` to hold the new file; ignore lists were updated accordingly.

## Code Pattern (Before vs. After)
```ts
// Before
import HtmlToMd from '@joplin/lib/HtmlToMd';
// ...
const htmlToMd = new HtmlToMd();
const htmlToMarkdown = (html: HTMLElement): string => {
	html = postprocessHtml(html);

	return htmlToMd.parse(html, { preserveColorStyles: true });
};

// After
import convertHtmlToMarkdown from './convertHtmlToMarkdown';
// ...
const htmlToMarkdown = (html: HTMLElement): string => {
	html = postprocessHtml(html);

	return convertHtmlToMarkdown(html);
};

// convertHtmlToMarkdown.ts
const TurndownService = require('@joplin/turndown');
const turndownPluginGfm = require('@joplin/turndown-plugin-gfm').gfm;

// Avoid using @joplin/lib/HtmlToMd here. HtmlToMd may cause several megabytes
// of additional JavaScript and supporting data to be included.
const convertHtmlToMarkdown = (html: string|HTMLElement) => {
	const turndown = new TurndownService(turndownOpts);
	turndown.use(turndownPluginGfm);
	turndown.remove('script');
	turndown.remove('style');
	return turndown.turndown(html);
};
```

## Takeaway
For code that runs in a WebView or injected bundle, import the smallest module that does the job; a convenient shared helper can drag in megabytes of unrelated code and data.
