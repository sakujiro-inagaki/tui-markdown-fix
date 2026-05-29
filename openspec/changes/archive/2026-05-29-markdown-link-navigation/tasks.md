## 1. Prep — relocate `inlines_plain_text` so `Ast.fix` can use it

- [x] 1.1 Move `inlines_plain_text` and its `_inline_plain` helper from `src/Yaynu/Markdown/Render/Inline.fix` into `src/Yaynu/Markdown/Ast.fix`. (Required so `collect_links`, which builds `LinkInfo.@display_text`, doesn't introduce a `Render → Ast` reverse import.)
- [x] 1.2 Update `src/Yaynu/Markdown/Render/Inline.fix` to import the new home (or rely on the existing `import Yaynu.Markdown.Ast;`).
- [x] 1.3 Update any other call sites (`Render/Table.fix`, `Render/Block.fix`) — `fix check` is the source of truth.
- [x] 1.4 Verify `fix check` is clean and `fix test` is still green before continuing.

## 2. Link enumeration (`markdown-link-enumeration` capability)

- [x] 2.1 In `src/Yaynu/Markdown/Ast.fix`, add `type LinkInfo = box struct { index : I64, url : String, display_text : String };`.
- [x] 2.2 Add `collect_links : Document -> Array LinkInfo` that walks blocks → inlines depth-first in source order per the contract in `design.md` D1 / spec D1. Container inlines (`emph`, `strong`, `strikethrough`, `link`) recurse before advancing; container blocks (`blockquote`, list items, table headers + rows + cells) recurse with the same rules; `image` does NOT contribute.
- [x] 2.3 Add `tests/YaynuMarkdownTest/LinkEnumerationTest.fix` covering each scenario in `specs/markdown-link-enumeration/spec.md`: empty document, multiple links across paragraphs, link in nested list item, link inside table cell, image excluded, `index == array position` invariant.
- [x] 2.4 Register the new test file in `fixproj.toml` (`[build.test].files`) and in `tests/YaynuMarkdownTest.fix`'s suite driver.

## 3. RenderOptions extension (`markdown-render` capability)

- [x] 3.1 In `src/Yaynu/Markdown/Render/Options.fix`, add `selected_link : Option I64` and `selected_link_style : Style` to the `RenderOptions` record.
- [x] 3.2 Update `RenderOptions::default` to set `selected_link = Option::none()` and `selected_link_style = Style::default.set_reverse(true)`.
- [x] 3.3 Confirm `fix check` is clean — every existing call site (`Test.fix`, examples, `MarkdownView`) constructs RenderOptions only via `default` or `.set_*`, so adding fields is non-breaking.

## 4. Counter threading through the renderer

- [x] 4.1 In `src/Yaynu/Markdown/Render/Inline.fix`, change `inlines_to_tokens : Array Inline -> Style -> MarkdownStyle -> Array InlineToken` to `inlines_to_tokens : Array Inline -> Style -> RenderOptions -> I64 -> (Array InlineToken, I64)` (last `I64` = `next_link_index`). Internal `_inline_to_tokens` becomes counter-aware too.
- [x] 4.2 In `_inline_to_tokens` for `link((kids, url))`: read `opts.@selected_link`. If `some(this_link_idx)`, additionally overlay `opts.@selected_link_style` onto the base style before recursing into `kids`. The link's own index is the counter's current value; increment AFTER recursion (so nested links inside this link get later indices).
- [x] 4.3 In `src/Yaynu/Markdown/Render/Block.fix`, change `render_block` and `render_blocks` to thread `start_idx` and return `next_idx`. Update `_render_paragraph`, `_render_heading`, `_render_blockquote`, `_render_list`, `_render_code_block` (no inlines, just thread through), `_render_thematic_break` (no inlines), `_render_html_block` (no inlines). For list items: walk items in order, threading the counter across each item's inner blocks. For blockquotes: thread through the recursive `render_blocks` call.
- [x] 4.4 In `src/Yaynu/Markdown/Render/Table.fix`, change `render_table_lines` to thread the counter. Walk headers left-to-right (each cell's inlines), then body rows left-to-right (each cell's inlines).
- [x] 4.5 In `src/Yaynu/Markdown/Render.fix`, keep the public entry points `render`, `render_string`, `estimate_height` at their existing signatures. Internally call `render_blocks(blocks, opts, width, 0)` and discard the final `next_idx`.
- [x] 4.6 Verify `fix check` is clean and `fix test` is still green (all existing tests should pass — the counter threading is invisible to callers).

## 5. Widget convenience (`markdown-widget` capability)

- [x] 5.1 In `src/Yaynu/Markdown/Widget.fix`, add `links : MarkdownView -> Array LinkInfo` returning `Ast::collect_links(v.@document)`.
- [x] 5.2 Verify `fix check` is clean.

## 6. Tests for selected_link rendering

- [x] 6.1 Add `tests/YaynuMarkdownTest/SelectedLinkTest.fix` covering the scenarios in `specs/markdown-render/spec.md`'s new requirements: out-of-range index degrades to none, `none` produces no highlight, `some(k)` highlights the k-th link by `collect_links` order, multi-row wrapped link gets the overlay on every row.
- [x] 6.2 The test asserts on the per-cell `Style.@reverse` field of the rendered `Buffer` (use `Buffer::get_cell` directly — the snapshot harness throws styles away so it can't verify this).
- [x] 6.3 Register the new test file in `fixproj.toml` and in the suite driver.

## 7. `examples/show.fix` — demonstrate TAB / Shift-TAB / Space

- [x] 7.1 Update `AppState` to also carry `selected : I64` (the currently-selected link index) and a precomputed `links : Array LinkInfo`.
- [x] 7.2 In the `update` handler, intercept `Key::tab()` (advance `selected`, wrapping at `links.get_size`) and a Shift-TAB binding if available (decrement, wrapping). On `Key::char(" ")`, if `selected` is in range, print the URL to stderr instead of advancing scroll — i.e. consume the space at the show-example level before forwarding to `handle_key`. Other keys still go through `handle_key`. (Shift-TAB skipped: term-fix 0.1.0 does not expose a Shift-TAB key constructor; stderr printing uses `debug_eprintln` since `TuiApp::update` is pure.)
- [x] 7.3 In the `view` handler, set `RenderOptions.@selected_link = some(s.@selected)` (when there are links) on the options used for rendering, so the selected link is visibly highlighted.
- [x] 7.4 Update the help-line text to mention TAB / Shift-TAB / Space.
- [x] 7.5 Compile-verify with `fix run -f examples/show.fix -s tui_shim -s term_shim -L .fixlang/deps/tui_0.1.2/c_src -L .fixlang/deps/term_0.1.0/c_src` (the no-arg invocation should still print the usage line).

## 8. Documentation

- [x] 8.1 Update `README.md`: add a "Link navigation" section describing `Ast::collect_links` / `Widget::links`, `RenderOptions.selected_link` + `selected_link_style`, and the TAB / Space integration pattern. Update the `RenderOptions` reference table.
- [x] 8.2 Add a CHANGELOG entry for this v0.2-track release (call it `0.2.0` — additive feature bump). Explicitly note OSC 8 was considered and deferred.

## 9. Acceptance

- [x] 9.1 `fix test` passes (including the two new test suites).
- [x] 9.2 `examples/show.fix` compiles and prints the usage line when run without args.
- [x] 9.3 For a fixture with multiple links (e.g. `tests/fixtures/snapshots/llm_response.md`), `Ast::collect_links` returns the expected URLs in source order. (Covered by `LinkEnumerationTest`.)
- [x] 9.4 Rendering with `selected_link = some(0)` highlights the first link and only the first link. (Covered by `SelectedLinkTest`.)
- [x] 9.5 README "Link navigation" section is accurate against the shipped code (no stale field names, no missing fields).
