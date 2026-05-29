# Changelog

## 0.2.0 — link navigation

Additive feature release: enumerate links and drive a "selected link"
highlight from a `RenderOptions` field. No breaking changes — every
v0.1 caller continues to compile and render identically without
modification.

### Added

- `Yaynu.Markdown.Ast::LinkInfo` (`box struct { index : I64, url :
  String, display_text : String }`) and `collect_links : Document ->
  Array LinkInfo`. Walks the document in source depth-first order;
  container inlines (`emph` / `strong` / `strikethrough` / `link`)
  recurse before advancing, so a link nested inside emphasis gets its
  own later index. `image` inlines are excluded.
  `count_links_in_inlines` is also exposed for renderers that need to
  advance a link counter without materialising the array.
- `Yaynu.Markdown.Widget::links : MarkdownView -> Array LinkInfo` —
  thin wrapper over `Ast::collect_links(v.@document)`.
- `Yaynu.Markdown.Render.Options::RenderOptions` gains two fields:
  `selected_link : Option I64` (defaults to `none()`) and
  `selected_link_style : Style` (defaults to
  `Style::default.set_reverse(true)`). When `selected_link` is
  `some(k)` and `k` is in range, the renderer overlays
  `selected_link_style` on top of the per-link palette for that
  link's cells (including across wrapped rows). Out-of-range indices
  silently degrade to `none` — no panic.
- `examples/show.fix` gains a full link-navigation demo:
  `Tab` / `Shift+Tab` cycle forward / backward through the links
  *currently visible* in the viewport (links scrolled off-screen are
  skipped, and the selection auto-clears when the selected link
  scrolls out); `Space` opens the selected URL in the default browser
  by shelling out to `open` (macOS) / `xdg-open` (Linux/BSD) via
  `system(3)`. `Esc` / `q` quit.
- Tests: new `LinkEnumerationTest` (source-DFS contract, empty doc,
  list / table-cell / image-excluded scenarios) and `SelectedLinkTest`
  (per-cell `Style.@reverse` assertions on the rendered `Buffer`,
  since the snapshot harness throws styles away).

### Changed

- `Yaynu.Markdown.Ast` now owns `inlines_plain_text` (relocated from
  `Render.Inline`). `Render.Inline` re-exports through the existing
  `import Yaynu.Markdown.Ast` so external callers see no change.
- Internal renderer signatures (`render_blocks`, `render_block`,
  `render_table_lines`, `inlines_to_tokens`) now thread a global link
  counter (`start_idx → next_idx`) so the index assigned by
  `collect_links` matches the renderer's. Public entry points
  (`Render::render`, `render_string`, `estimate_height`,
  `Frame::render_markdown_view`) keep their existing signatures.
- Dependency bump: `tui = "0.2.0"` (transitively `term = "0.2.0"`).
  Required for `Key::back_tab` (Shift+Tab as `ESC[Z`) and the fix to
  tui-fix's spurious `Key::escape` injection for unknown CSI
  sequences. Update the `library_paths` entries in your
  `fixproj.toml` from `.fixlang/deps/tui_0.1.2/c_src` /
  `.fixlang/deps/term_0.1.0/c_src` to the `_0.2.0` paths to match.

### Considered, not shipped

- OSC 8 clickable hyperlinks. Originally on the v0.2 wish list, dropped
  after weighing it against the actual demand: `collect_links` +
  `selected_link` is enough to ship the navigation UX entirely on the
  caller side without a `tui-fix` upstream change. May revisit if a
  concrete need surfaces.
- A general per-link styling callback `(I64, Style) -> Style`.
  `selected_link` covers the real use case.

## 0.1.0 — initial release

First public release of `tui-markdown-fix` under the temporary
`Yaynu.Markdown.*` namespace (a planned `TuiMarkdown.*` rename ships
with the next major version).

### Added

- AST types in `Yaynu.Markdown.Ast`: `Document`, `Block`, `Inline`,
  `CodeBlock`, `MarkdownList`, `ListItem`, `Table`, `Alignment`.
- `Yaynu.Markdown.Parser::parse : String -> Document` covering ATX
  headings, paragraphs, fenced & indented code blocks, blockquotes
  (with nesting), unordered / ordered / task lists (with nesting),
  thematic breaks, GFM tables (with `:---:` alignment encoding),
  HTML block passthrough, and inline emphasis / strong /
  strikethrough / inline code / links / images / `http(s)://`
  autolinks / angle-bracket autolinks / hard breaks / raw HTML
  passthrough.
- `Yaynu.Markdown.Style::MarkdownStyle` record with three presets:
  `default`, `dark`, `monochrome`.
- `Yaynu.Markdown.Render::render` / `render_string` /
  `estimate_height`, plus `RenderOptions` and `TableBorderStyle`
  (`none` / `light` / `heavy` / `ascii`).
- `Yaynu.Markdown.Widget::MarkdownView` with `new`, `with_options`,
  `with_scroll`, `total_height`, `handle_key`, plus
  `Frame::render_markdown_view` (drop-in for `Frame::render_paragraph`).
- `Yaynu.Markdown.Test::buffer_debug_string` + `render_to_string`
  helpers used by the snapshot test harness.
- Example programs `examples/all_features.fix`,
  `examples/llm_response.fix`, `examples/show.fix` (interactive
  pager).

### Known limitations

- No syntax highlighting (extension point reserved for v0.2).
- Links render their display text only — the URL is dropped.
- Images render as the literal text `[image: <alt>]`.
- No streaming / incremental parsing.
- Inline parsing is greedy outside-in; CommonMark delimiter-run
  subtleties are not modelled.
- See [`README.md`](README.md#not-supported-v01-limitations) for the
  complete v0.1 limitations list.
