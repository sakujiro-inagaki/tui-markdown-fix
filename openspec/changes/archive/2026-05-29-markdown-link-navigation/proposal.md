## Why

`Yaynu.Markdown` v0.1 renders links as styled text only — the URL is dropped and there is no way for a TUI to (a) enumerate the links in the document or (b) tell the renderer "highlight link N as currently selected." This blocks the natural TUI pattern of "TAB cycles links, Space opens the selected one." Adding these two primitives now keeps the public API additive — no v0.1 caller breaks — and they're enough to ship the navigation UX end-to-end on the caller side without further library changes.

## What Changes

- Add `Yaynu.Markdown.Ast::collect_links : Document -> Array LinkInfo` (pure AST query) returning every link in source-DFS order, with index, URL, and plain-text display. Add `Yaynu.Markdown.Widget::links : MarkdownView -> Array LinkInfo` as the widget-side convenience.
- Add two fields to `RenderOptions`: `selected_link : Option I64` (source-DFS index of the link to highlight, `none` = nothing selected) and `selected_link_style : Style` (overlaid on top of the per-link palette style; default = reverse video). The renderer threads a global link counter so the index assigned by `collect_links` matches the one used for `selected_link`.
- **Non-goals (deferred):**
  - OSC 8 clickable hyperlinks. (Originally on the v0.2 wish list, dropped after weighing the tui-fix upstream surface area against the actual demand. The same `LinkInfo` enumeration unlocks `Space`-to-open on the caller side without OSC 8.)
  - A general per-link styling callback `(I64 -> Style -> Style)` — `selected_link` covers the actual use case; revisit only if a real multi-link-styling need surfaces.
  - Image links — the `image` inline still expands to `[image: alt]` and is not enumerable as a navigable link.
  - Auto-scroll to bring the selected link on-screen.

## Capabilities

### New Capabilities

- `markdown-link-enumeration`: pure AST query `collect_links : Document -> Array LinkInfo` plus a `LinkInfo` shape (`index : I64`, `url : String`, `display_text : String`). Used by callers to drive TAB navigation and `Space`-to-open without re-walking the AST themselves.

### Modified Capabilities

- `markdown-render`: extend `RenderOptions` with `selected_link` and `selected_link_style`; document the source-DFS link-index contract that `collect_links` and the renderer share.
- `markdown-widget`: add `MarkdownView::links` convenience returning `collect_links(v.@document)`, and document the recommended TAB / Space integration pattern (caller maintains `selected : I64`, sets it on `RenderOptions.selected_link` before each render).

## Impact

- **New code** in `src/Yaynu/Markdown/Ast.fix` (`LinkInfo` type + `collect_links`), `src/Yaynu/Markdown/Render/Options.fix` (two new `RenderOptions` fields), `src/Yaynu/Markdown/Render/Inline.fix` (link-counter threading + `selected_link_style` overlay), `src/Yaynu/Markdown/Render/Block.fix` and `src/Yaynu/Markdown/Render/Table.fix` (counter-aware variants that thread the link index through `render_block` / `render_blocks` / `render_table_lines` for every block type that contains inlines: paragraph, heading, blockquote inner, list item inner, table cell), `src/Yaynu/Markdown/Widget.fix` (`MarkdownView::links`).
- **No tui-fix changes.** All work is contained in `tui-markdown-fix`. The dep pin stays at `tui = "0.1.2"`.
- **No breaking changes.** `RenderOptions::default` keeps `selected_link = none()` so existing callers see identical output. `Ast::collect_links` and `Widget::links` are pure additions.
- **Tests:** new unit suite for `collect_links` (DFS order, links in lists / blockquotes / tables, image is not a link, empty doc) and a `selected_link` rendering test that asserts the highlight lands on the correct link by `collect_links` index.
- **Yaynu integration:** still deferred (Phase 1 of Yaynu has no TUI). The new API is shaped to drop in alongside `MarkdownView` once Yaynu has a key router. `examples/show.fix` gains a TAB / Shift-TAB / Space demo so the UX is verifiable on a representative document.
