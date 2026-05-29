## Why

Yaynu displays LLM responses in a TUI, and those responses are almost always GitHub Flavored Markdown (headings, lists, fenced code, tables, inline emphasis). Rendering them as raw text — or even with the existing `Paragraph` widget — produces noisy, hard-to-scan output. We need a markdown-aware view so that LLM output (the dominant content type) is legible inside Yaynu's terminal UI.

## What Changes

- Add a markdown rendering pipeline (parse → AST → render) under the `Yaynu.Markdown.*` namespace, living inside the current Yaynu codebase. Future extraction to a standalone `tui-markdown-fix` library is in scope as a non-goal-for-now design constraint, but **not** done in this change.
- Introduce a parser for GFM's common subset: paragraphs, ATX headings (1–6), unordered/ordered/task lists (with nesting), fenced and indented code blocks, blockquotes (with nesting), thematic breaks, GFM tables, plus inline emphasis/strong/strikethrough/inline-code/link/autolink/image/hard-break/raw-HTML (passthrough).
- Introduce a renderer that draws an AST into a `tui-fix` `Buffer` within a given `Rect`, with configurable styling (`MarkdownStyle`) and options (`RenderOptions`: wrap, indent, border, table border style, etc.).
- Introduce a `MarkdownView` widget that wraps document + options + scroll state and plugs into Yaynu's existing `Frame` rendering, replacing the bare `Paragraph` path for LLM responses.
- Add tests: parser unit tests, renderer unit tests against a debug-stringified buffer, and snapshot fixtures (including Japanese and a representative LLM-response sample). Add a `Buffer::debug_string`-style helper used by tests (in this library, not upstream tui-fix).
- Document scope and v0.1 limitations in README (no syntax highlighting; naive inline parsing; links render but are not clickable; image alt-text only).
- **Non-goals (deferred):** syntax highlighting, footnotes, math, mentions, emoji shortcodes, `<details>` folding, rich `<br>`, OSC 8 clickable links. The API is shaped so a syntax highlighter can be injected later without breaking changes.

## Capabilities

### New Capabilities
- `markdown-ast`: Data types representing a parsed GFM document (`Document`, `Block`, `Inline`, `MarkdownList`, `ListItem`, `Table`, `Alignment`, `CodeBlock`).
- `markdown-parser`: Pure function `parse : String -> Document` covering the GFM subset listed above, with documented "naive" inline-parsing behavior.
- `markdown-render`: AST-to-Buffer rendering with configurable `MarkdownStyle` and `RenderOptions`, including paragraph wrapping, table layout, blockquote/list nesting, code-block decoration, and East Asian Width handling via `tui-fix`.
- `markdown-widget`: `MarkdownView` widget integrating with Yaynu's TUI frame, providing scrolling and key handling (arrows, PageUp/Down, Home/End) and a `total_height` query for scroll bounds.

### Modified Capabilities
<!-- None: this change introduces new capabilities only; existing Yaynu specs are not modified at the requirement level. The integration point (using `MarkdownView` in place of `Paragraph` for LLM output) is an implementation choice, not a spec-level requirement change. -->

## Impact

- **New code** under `src/Yaynu/Markdown/` (facade, AST, parser, style, render, widget) plus tests under `tests/` and fixtures under `tests/fixtures/snapshots/`. Plus optional `examples/` (`show.fix`, `all_features.fix`, `llm_response.fix`).
- **Dependencies**: relies on `tui-fix` (`Buffer`, `Style`, `Width`) and `Std` only. Does not add `Minilib` or any other dependency. Confirms tui-fix is already a project dependency before merging (it is, since Yaynu's existing TUI is built on it).
- **Affected Yaynu code**: the LLM-response rendering path is migrated from `Paragraph` to `MarkdownView`. Other Yaynu screens are untouched.
- **Performance**: parsing/rendering happens per redraw of the response panel. For typical LLM responses (≤ a few KB) this is acceptable; if a regression surfaces, memoize the parsed `Document` on the source string in `MarkdownView`.
- **Future**: API is intentionally extensible so v0.2 can add `RenderOptions.syntax_highlighter : Option SyntaxHighlighter` and a separate `tui-syntax-fix` library can plug in without changes here. Namespace `Yaynu.Markdown.*` will be renamed to `TuiMarkdown.*` when extracted; this is called out in the README.
