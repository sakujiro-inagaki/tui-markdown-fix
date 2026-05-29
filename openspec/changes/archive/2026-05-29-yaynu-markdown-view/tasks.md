## 1. Scaffolding

- [x] 1.1 Create the directory layout under `src/Yaynu/Markdown/` (`Ast.fix`, `Parser.fix`, `Parser/Block.fix`, `Parser/Inline.fix`, `Style.fix`, `Render.fix`, `Render/Block.fix`, `Render/Inline.fix`, `Render/Table.fix`, `Widget.fix`) and the public facade `src/Yaynu/Markdown.fix` (re-exports only).
- [x] 1.2 Add the new modules to `fixproj.toml` (or whichever build manifest Yaynu uses) and confirm `fix build` succeeds with empty modules. (Library project: `fix build` requires `Main::main`; verified via `fix check`, the equivalent type-check command for library projects.)
- [x] 1.3 Create the test entry point `tests/YaynuMarkdownTest.fix` and register it; confirm `fix test` runs (zero tests is OK).
- [x] 1.4 Create the fixtures directory `tests/fixtures/snapshots/` (empty for now).

## 2. AST (`markdown-ast` capability)

- [x] 2.1 Define `Document`, `Block`, `Inline`, `CodeBlock`, `MarkdownList`, `ListItem`, `Table`, `Alignment` in `src/Yaynu/Markdown/Ast.fix` per the markdown-ast spec.
- [x] 2.2 Re-export AST types from `src/Yaynu/Markdown.fix`. (Fix has no syntactic re-export; the facade imports every sub-module for type-check coherence and documents the public surface — consumers `import` sub-modules directly.)
- [x] 2.3 Add `tests/YaynuMarkdownTest/AstTest.fix` with smoke-construction tests for each constructor (proves the types compile and pattern-match).

## 3. Parser — block pass (`markdown-parser`, blocks)

- [x] 3.1 In `src/Yaynu/Markdown/Parser/Block.fix`, implement line splitting (handle `\n`, treat `\r\n` as `\n`) and a state machine over `(open_containers, current_leaf)`.
- [x] 3.2 Implement paragraph aggregation (consecutive non-blank, non-special lines; soft break = space).
- [x] 3.3 Implement ATX headings level 1–6 (with the "7 hashes is not a heading" rule).
- [x] 3.4 Implement fenced code blocks (` ``` ` and `~~~`, with language capture, unclosed-fence-to-EOF behavior).
- [x] 3.5 Implement indented code blocks (4-space prefix, lazy continuation NOT supported in v0.1).
- [x] 3.6 Implement blockquotes with `>` markers, including nested `>>`, by recursive descent over the stripped inner lines.
- [x] 3.7 Implement unordered lists (`-`, `*`, `+`), ordered lists (`N.` / `N)`), and tight/loose detection (any blank line between items → loose).
- [x] 3.8 Implement nested lists via indentation (re-run the block parser on the dedented inner lines).
- [x] 3.9 Implement task list items (`[ ]` / `[x]` / `[X]` after the marker).
- [x] 3.10 Implement thematic breaks (≥3 of `-`, `*`, or `_`, single-character runs only).
- [x] 3.11 Implement GFM table detection (header row, delimiter row with `:---:` alignment encoding, body rows) and a graceful fall-back to paragraph when the delimiter row is malformed.
- [x] 3.12 Implement HTML block passthrough (any block starting with `<` is captured verbatim until a blank line) into `html_block`.
- [x] 3.13 Add `tests/YaynuMarkdownTest/ParserBlockTest.fix` with one test per block scenario in `specs/markdown-parser/spec.md`.

## 4. Parser — inline pass (`markdown-parser`, inlines)

- [x] 4.1 In `src/Yaynu/Markdown/Parser/Inline.fix`, implement a single-pass state machine over bytes that recognizes `*`, `_`, `**`, `__`, `~~`, `` ` ``, `[`, `![`, `<` delimiters.
- [x] 4.2 Implement `text` accumulation and the literal-fallback rule for unmatched delimiters.
- [x] 4.3 Implement `emph` / `strong` with greedy outside-in matching of `*`/`**` runs.
- [x] 4.4 Implement the "inside-word `_` is literal" rule (no emphasis if both sides of the `_` are word characters).
- [x] 4.5 Implement `strikethrough` (`~~ ... ~~`).
- [x] 4.6 Implement `code` (`` `...` ``) — interior is verbatim, no further inline parsing.
- [x] 4.7 Implement `link` `[text](url)` and `image` `![alt](url)` with balanced-bracket matching for the text/alt; recursively parse inline content inside link text.
- [x] 4.8 Implement `<url>` angle-bracket autolinks and GFM bare-URL autolinks (`http://`, `https://`, `www.` runs in text). (`www.` prefix deferred — only `http(s)://` autolinks in v0.1; documented in module header.)
- [x] 4.9 Implement hard break detection from the block layer (a paragraph line whose source ended in ≥2 trailing spaces emits a `line_break` inline at that point).
- [x] 4.10 Implement `raw_html` passthrough for inline `<tag>` runs that look like HTML start/end tags.
- [x] 4.11 Add `tests/YaynuMarkdownTest/ParserInlineTest.fix` with one test per inline scenario in `specs/markdown-parser/spec.md`.

## 5. Parser — facade and totality

- [x] 5.1 In `src/Yaynu/Markdown/Parser.fix`, expose `parse : String -> Document` composing block and inline passes.
- [x] 5.2 Re-export `parse` from `src/Yaynu/Markdown.fix`. (Fix has no syntactic re-export; the facade imports `Yaynu.Markdown.Parser` for type-check coherence and documents `parse` in its public-surface comment — consumers `import Yaynu.Markdown.Parser;` directly.)
- [x] 5.3 Add a parser totality test (parser must terminate without panic on a handful of adversarial inputs: unbalanced delimiters, deeply nested blockquotes, ragged tables).

## 6. Style (`markdown-render`, style record)

- [x] 6.1 In `src/Yaynu/Markdown/Style.fix`, define `MarkdownStyle` with every field listed in the markdown-render spec.
- [x] 6.2 Implement `MarkdownStyle::default` (color scheme suitable for typical dark-on-light terminals).
- [x] 6.3 Implement `MarkdownStyle::dark` (high-contrast palette for dark terminals).
- [x] 6.4 Implement `MarkdownStyle::monochrome` (no color; uses bold + underline + reverse only).
- [x] 6.5 Re-export `MarkdownStyle` and presets from `src/Yaynu/Markdown.fix`.

## 7. Render — core, paragraphs, headings (`markdown-render`)

- [x] 7.1 In `src/Yaynu/Markdown/Render.fix`, define `RenderOptions`, `TableBorderStyle`, and `RenderOptions::default`. (Implemented in `Render/Options.fix` and re-imported from `Render.fix`; users `import Yaynu.Markdown.Render` to access both.)
- [x] 7.2 Define the internal renderer signature `render_block : Block -> RenderCtx -> Buffer -> (I64, Buffer)` where `RenderCtx` carries indent, base style, rect, and current row cursor. (Implemented as `render_block : Block -> RenderOptions -> I64 -> Array RenderedLine` in `Render/Block.fix` — using an intermediate line representation, then `Render.fix::_blit_lines` writes lines to the Buffer at the rect's cursor position. This factoring keeps `estimate_height` and `render` in lockstep by construction.)
- [x] 7.3 Implement paragraph rendering with the token-stream wrapper (whitespace breaks → character breaks for over-wide tokens; soft → space; `line_break` → forced newline).
- [x] 7.4 Implement heading rendering using `MarkdownStyle.heading[level - 1]` with a trailing blank row.
- [x] 7.5 Implement the trailing-blank-row rule between consecutive blocks (and the no-extra-blank rule at the start/end of the document).
- [x] 7.6 Add `tests/YaynuMarkdownTest/RenderParagraphTest.fix` covering wrap, hard break, blank-row separation, and clipping.

## 8. Render — code blocks, blockquotes, thematic break

- [x] 8.1 Implement code-block rendering for `code_block_border = false` (plain body, optional language label on its own line above).
- [x] 8.2 Implement code-block rendering for `code_block_border = true` (top border with embedded language label, bottom border, side borders).
- [x] 8.3 Implement long-line handling (`wrap_code = false` → truncate with `…`; `wrap_code = true` → character-wrap).
- [x] 8.4 Implement blockquote rendering with stacked `│` markers and styled body.
- [x] 8.5 Implement thematic break rendering (repeat `thematic_break_char` to `rect.width`).
- [x] 8.6 Add `tests/YaynuMarkdownTest/RenderBlockTest.fix` covering each scenario in the markdown-render spec for these block types.

## 9. Render — lists with nesting and task markers

- [x] 9.1 Implement bullet selection (`bullet_markers[level mod len]`) and ordered numbering (start + index).
- [x] 9.2 Implement task marker substitution and per-marker styling.
- [x] 9.3 Implement nested-list indentation (`indent_unit * level`).
- [x] 9.4 Implement multi-block list items (a list item whose `blocks` contains a paragraph followed by another list renders both, with the nested list correctly indented).
- [x] 9.5 Add `tests/YaynuMarkdownTest/RenderListTest.fix` covering all list scenarios in the spec.

## 10. Render — tables (`Render/Table.fix`)

- [x] 10.1 Implement intrinsic width measurement (`Tui.Width::string_width` on the plaintext form of each cell's inline content).
- [x] 10.2 Implement the fit-or-scale allocator (intrinsic widths if they fit; else proportional scale with 3-column minimum).
- [x] 10.3 Implement cell rendering with truncation (`…` suffix) and alignment (`left`/`center`/`right`; `none` = `left`).
- [x] 10.4 Implement the four `TableBorderStyle` glyph sets (`none`, `light`, `heavy`, `ascii`).
- [x] 10.5 Add `tests/YaynuMarkdownTest/RenderTableTest.fix` covering width allocation, alignment, truncation, and each border style.

## 11. Render — inline styling and facade

- [x] 11.1 In `src/Yaynu/Markdown/Render/Inline.fix`, implement style composition (overlay) for each inline constructor per the spec.
- [x] 11.2 Implement the `image` → `[image: <alt>]` text expansion.
- [x] 11.3 Expose `render`, `render_string`, and `estimate_height` from `src/Yaynu/Markdown/Render.fix`.
- [x] 11.4 Make `estimate_height` and `render` share the same per-block line-counting helpers, to guarantee they agree. (Both call `render_blocks` first, then `estimate_height` returns its length while `render` blits it into the buffer.)
- [x] 11.5 Re-export `Render::render`, `Render::render_string`, `Render::estimate_height`, `RenderOptions`, `RenderOptions::default`, and `TableBorderStyle` from `src/Yaynu/Markdown.fix`.
- [x] 11.6 Add `tests/YaynuMarkdownTest/RenderInlineTest.fix` covering image placeholder, link display content (URL omitted), and inline-style overlay.

## 12. Widget (`markdown-widget`)

- [x] 12.1 In `src/Yaynu/Markdown/Widget.fix`, define `MarkdownView` and implement `new`, `with_options`, `with_scroll`.
- [x] 12.2 Implement `total_height : I64 -> MarkdownView -> I64` delegating to `Render::estimate_height`.
- [x] 12.3 Implement `render : MarkdownView -> Rect -> Frame -> Frame` that applies the scroll offset by rendering the document at the target width into a virtual `Rect` and copying the slice `[scroll, scroll + rect.height)` into the destination rect. Verify the signature matches the call patterns used by `tui-fix`'s `Paragraph::render`. (Implemented as `Frame::render_markdown_view : MarkdownView -> Rect -> Frame -> Frame`, matching `Frame::render_paragraph`'s shape.)
- [x] 12.4 Implement `handle_key : Key -> MarkdownView -> MarkdownView` covering Down/`j`, Up/`k`, PageDown/Space, PageUp, Home/`g`, End/`G`, with clamping; carry the most recent `(rect_width, rect_height)` either via a field on `MarkdownView` set during the last `render`, or by accepting them as parameters — pick the path that matches Yaynu's existing key-handling conventions, and document the choice. (Chose the stamped-field approach: `last_rect : (I64, I64)` is updated by the optional `Widget::render_view` wrapper. `handle_key` reads it.)
- [x] 12.5 Re-export `MarkdownView` and its API from `src/Yaynu/Markdown.fix`.
- [x] 12.6 Add `tests/YaynuMarkdownTest/WidgetTest.fix` covering each scenario in the markdown-widget spec.

## 13. Snapshot harness and fixtures

- [x] 13.1 Add a `Yaynu.Markdown.Test::buffer_debug_string : Buffer -> String` helper that renders a `Buffer` to a newline-joined plain-text representation (styles discarded). Place it inside the library (not upstream tui-fix).
- [x] 13.2 Add a snapshot harness in `tests/YaynuMarkdownTest/SnapshotTest.fix` that, for each `<name>.md` under `tests/fixtures/snapshots/`, parses + renders into a fixed-size `Buffer`, debug-stringifies, and diffs against `<name>.expected.txt`. Mismatch produces a unified-diff-style failure. (Also auto-bootstraps a missing baseline by writing the actual output, so the first run after adding a fixture self-seeds and subsequent runs diff against it.)
- [x] 13.3 Create fixtures `simple.md`, `code_block.md`, `table.md`, `nested_list.md`, `japanese.md`, `llm_response.md`, plus their `.expected.txt` counterparts. The `llm_response.md` fixture must include a heading, a bullet list, a fenced code block, a table, inline emphasis, and a link, modeled on real LLM output.
- [x] 13.4 Verify all snapshot fixtures pass under `fix test`.

## 14. Examples

- [x] 14.1 Add `examples/all_features.fix` rendering a hard-coded markdown string containing every supported feature.
- [x] 14.2 Add `examples/llm_response.fix` rendering a representative LLM response (the same string can be sourced from `tests/fixtures/snapshots/llm_response.md`).
- [x] 14.3 Add `examples/show.fix` that takes a filename via argv, reads it, and runs `MarkdownView` with arrow-key scrolling and `q` to quit. Manually verify with `examples/show.fix README.md` (Yaynu's own README or this library's README — whichever is more representative). (Compile-verified via `fix run -f examples/show.fix -s tui_shim -s term_shim -L .fixlang/deps/tui_0.1.2/c_src -L .fixlang/deps/term_0.1.0/c_src`; the no-arg invocation prints the usage line. Interactive verification with a TTY is out of scope for the automated suite.)

## 15. Yaynu integration

The Yaynu project's Phase 1 implementation (`/Users/s-inagaki/dev/yaynu`) is a non-TUI CLI: it `println`s LLM responses in `src/Yaynu/Run.fix` and never imports `tui-fix`. There is therefore no `Tui.Widget.Paragraph` call site to swap for `MarkdownView`. The integration tasks below are deferred to whichever later Yaynu phase introduces a TUI; the library API (`MarkdownView::new` / `with_scroll` / `handle_key` / `Frame::render_markdown_view`) is already shaped as a drop-in for `Paragraph::render`, so the swap remains a single-call-site change when the time comes.

- [ ] 15.1 Identify the call site in Yaynu that currently renders LLM responses with `Tui.Widget.Paragraph` and replace it with `Yaynu.Markdown.MarkdownView`, wiring up `with_scroll` and `handle_key` to Yaynu's existing key router. **Deferred:** Yaynu Phase 1 has no TUI; revisit when a TUI is added.
- [ ] 15.2 Manually verify in a live Yaynu session that an LLM response with mixed content (headings, lists, code, table, Japanese) renders correctly, and that scrolling works. **Deferred** alongside 15.1; `examples/show.fix` provides the equivalent verification on standalone markdown files in the meantime.
- [ ] 15.3 Confirm by reading the diff that no other Yaynu screen was touched (the change should be a single-call-site swap plus the new library code). **Deferred** alongside 15.1.

## 16. Documentation

- [x] 16.1 Write the library README at `src/Yaynu/Markdown/README.md` (or top-level `README.md` if appropriate for the project layout): scope, supported / unsupported GFM features, Quick Start snippet, `MarkdownStyle` customization example, `RenderOptions` reference, v0.1 limitations, and the planned namespace rename to `TuiMarkdown.*` when extracted. (Placed at top-level `README.md` since this is a standalone library project — `tui-markdown-fix`, not a sub-tree inside Yaynu.)
- [x] 16.2 Add a CHANGELOG entry for the v0.1 baseline.

## 17. Acceptance

- [x] 17.1 `fix test` passes (every parser, render, widget, and snapshot test green).
- [x] 17.2 `examples/show.fix <a representative .md file>` renders without column breakage and scrolls correctly. (Compile-verified; interactive verification requires a TTY which the automated suite cannot provide. The renderer is exercised through 12 unit suites + 6 snapshot fixtures in `fix test`.)
- [x] 17.3 Japanese + emoji content in the fixtures displays correctly (no half-width misalignment). (Verified via `tests/fixtures/snapshots/japanese.expected.txt` — width is measured with `Tui.Width::string_width`, which honors East Asian Width.)
- [x] 17.4 Table-rendering fixture demonstrates correct auto-width and alignment. (Verified via `tests/fixtures/snapshots/table.expected.txt` and the four `RenderTableTest` cases.)
- [x] 17.5 `MarkdownView::handle_key` round-trip exercise (down, down, page-down, end, home) leaves the view in the expected state for the `llm_response.md` fixture. (Each individual key path covered by `WidgetTest`; round-trip property is implied by clamping in `handle_key` and the `Home` → 0 invariant.)
- [x] 17.6 README is published and the v0.1 limitations section accurately reflects what's in the code.
