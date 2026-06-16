# EPUB TOC QuickPick Design

## Goal

Add a native VS Code chapter table-of-contents jump feature for EPUB books. The first version only supports EPUB files and uses a `QuickPick` command instead of adding a Webview or persistent sidebar view.

## Scope

In scope:

- Add a command named `zloveread.showToc` with the title `显示章节目录`.
- Parse EPUB chapter metadata during EPUB loading.
- Store chapter labels and their corresponding `processedContent` positions in memory for the current book.
- Show the current EPUB chapters in a `QuickPick`.
- Jump to the selected chapter, update the status bar, and save reading progress.

Out of scope:

- TXT chapter detection.
- Persistent TreeView chapter panel.
- Webview-based chapter UI.
- Nested chapter rendering beyond preserving optional level metadata.
- Cross-session persisted TOC cache.

## User Experience

When the user runs `显示章节目录` while reading an EPUB, VS Code shows a chapter list with `QuickPick`.

Each item shows:

- `label`: chapter title.
- `description`: approximate progress percentage, derived from the chapter start position and total processed lines.

When the user selects a chapter:

- `currentPosition` is set to the chapter start line.
- The status bar updates immediately.
- Reading progress is saved.

If the current book is not an EPUB, the command shows `当前书籍不支持目录跳转`.

If the EPUB has no readable chapter metadata, the command shows `未找到章节目录`.

## Data Model

Extend `BookContent` with optional chapter metadata:

```ts
interface ChapterInfo {
  id?: string;
  label: string;
  position: number;
  level?: number;
}

interface BookContent {
  content: string[];
  format: 'txt' | 'epub';
  totalWords: number;
  processedContent: string[];
  chapters?: ChapterInfo[];
}
```

The `position` field is always an index into `processedContent`, not a source chapter index or character offset.

## EPUB Loading

The existing EPUB loader reads `epub.flow` and appends each chapter's cleaned text to `allContent`. The TOC implementation will keep this load path but add deterministic chapter-position mapping.

The loader will:

1. Build a lookup table from EPUB metadata to human-readable chapter titles.
2. Read chapters in flow order.
3. Before appending each chapter's text, record the current source character offset.
4. Append cleaned text to the full source text buffer.
5. After all content is processed, convert each recorded source offset into a `processedContent` line index.

The offset-to-position conversion will use the configured `lineLength`. The computed position must be clamped to `0..processedContent.length - 1`.

If a chapter has no title in EPUB metadata, use a fallback label such as `章节 1`.

## Command Registration

Add `zloveread.showToc` to:

- `package.json` command contributions.
- `activate()` command registration.
- `context.subscriptions`.

No default keybinding will be added in the first version to avoid conflicts with user shortcuts.

## Error Handling

The command handles these cases:

- No book loaded: show `请先选择一本电子书`.
- Current book is TXT: show `当前书籍不支持目录跳转`.
- EPUB loaded but chapter list is empty: show `未找到章节目录`.
- Chapter position is stale or out of range: clamp to a valid line before jumping.

EPUB parsing failures continue to show the existing load failure message.

## Testing

Add focused unit coverage where practical:

- offset-to-position conversion with normal, zero, and out-of-range offsets.
- chapter position clamp behavior.
- fallback chapter labels when metadata title is missing.

The existing VS Code extension test remains as the integration entry point. Full EPUB fixture-based integration testing can be added later, but is not required for this first slice.

## Migration

No persisted data migration is required. The TOC is derived at load time and kept only in memory for the current book.

## Future Extensions

This design leaves room for:

- Adding TXT chapter detection later.
- Moving the chapter list into a TreeView alongside reading history.
- Persisting parsed TOC metadata if EPUB parsing becomes a performance issue.
