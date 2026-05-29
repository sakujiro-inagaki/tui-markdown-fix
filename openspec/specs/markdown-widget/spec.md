# markdown-widget Specification

## Purpose

TBD - created by archiving change yaynu-markdown-view. Update Purpose after archive.
## Requirements
### Requirement: MarkdownView construction and updaters

The system SHALL provide a `MarkdownView` record holding `document : Document`, `options : RenderOptions`, and `scroll : I64`, together with:
- `MarkdownView::new : String -> MarkdownView` — parses the input string once and stores the resulting `Document`, with `options = RenderOptions::default` and `scroll = 0`.
- `MarkdownView::with_options : RenderOptions -> MarkdownView -> MarkdownView` — returns a copy with `options` replaced.
- `MarkdownView::with_scroll : I64 -> MarkdownView -> MarkdownView` — returns a copy with `scroll` replaced (no clamping at this layer; clamping happens inside `handle_key` and at render time).

#### Scenario: new parses the source once

- **WHEN** `MarkdownView::new(src)` is called
- **THEN** the result's `@document` SHALL equal `Parser::parse(src)`, computed exactly once during the call

#### Scenario: with_options preserves document and scroll

- **WHEN** `v2 = v.with_options(opts2)` is computed
- **THEN** `v2.@document` SHALL equal `v.@document` and `v2.@scroll` SHALL equal `v.@scroll`

### Requirement: Total height query

The system SHALL provide `MarkdownView::total_height : I64 -> MarkdownView -> I64` that, given a target display width, returns the total number of rows the view's document would occupy when rendered with the view's current options at that width. It SHALL be equivalent to `Render::estimate_height(v.@document, v.@options, width)`.

#### Scenario: Equivalence to estimate_height

- **WHEN** `total_height(w, v)` is called
- **THEN** the result SHALL equal `Render::estimate_height(v.@document, v.@options, w)`

### Requirement: Render integration with Frame

The system SHALL provide `MarkdownView::render : MarkdownView -> Rect -> Frame -> Frame` that writes the view's document into the given `Rect` of the `Frame`'s underlying `Buffer`, offsetting the document by `scroll` rows (so the row at index `scroll` of a hypothetical unclipped render appears at the top of the rect). The signature SHALL match the conventions of `tui-fix`'s `Paragraph::render` so a caller can swap one for the other.

#### Scenario: Scroll offset shifts content

- **WHEN** `v.with_scroll(k).render(rect, frame)` is invoked and `k > 0`
- **THEN** the first row of the rect SHALL display what would be row `k` of `v.with_scroll(0).render(rect_with_unlimited_height, frame)`

#### Scenario: Scroll past end shows blank rows

- **WHEN** `scroll >= total_height(rect.width, v)`
- **THEN** the rect SHALL be filled with blank cells in the document's base background style, and no error SHALL be raised

### Requirement: Key handling

The system SHALL provide `MarkdownView::handle_key : Key -> MarkdownView -> MarkdownView` that updates `scroll` for the following keys, clamping the result to `[0, max(0, total_height(rect_width, v) - rect_height)]` using the most recently known `rect_width` and `rect_height` (passed implicitly through configuration or recomputed by the caller):

- Arrow Down / `j`: scroll += 1
- Arrow Up / `k`: scroll -= 1
- PageDown / `Space`: scroll += `rect_height` (one viewport)
- PageUp: scroll -= `rect_height`
- Home / `g`: scroll = 0
- End / `G`: scroll = clamp upper bound

Keys not in this set SHALL leave the view unchanged.

#### Scenario: Down arrow increments scroll

- **WHEN** `handle_key(arrow_down, v)` is invoked and `v.@scroll < upper_bound`
- **THEN** the returned view's `@scroll` SHALL be `v.@scroll + 1`

#### Scenario: Up arrow does not go below 0

- **WHEN** `handle_key(arrow_up, v)` is invoked and `v.@scroll == 0`
- **THEN** the returned view's `@scroll` SHALL remain `0`

#### Scenario: Home jumps to top

- **WHEN** `handle_key(home, v)` is invoked
- **THEN** the returned view's `@scroll` SHALL be `0`

#### Scenario: Unhandled key is identity

- **WHEN** `handle_key(k, v)` is invoked for any `k` not in the documented set
- **THEN** the returned view SHALL be structurally equal to `v`

### Requirement: Drop-in compatibility with Paragraph

The shape and arity of `MarkdownView::render` SHALL be compatible with the call patterns Yaynu currently uses for `Tui.Widget.Paragraph`, so substituting a `MarkdownView` for a `Paragraph` at a render call site requires no changes to the surrounding Frame plumbing.

#### Scenario: Swap at call site

- **WHEN** a Yaynu screen replaces `paragraph.render(rect, frame)` with `markdown_view.render(rect, frame)`
- **THEN** the program SHALL type-check without changes to the `frame` value's type or the `rect`'s type

### Requirement: MarkdownView::links convenience

The system SHALL provide `Yaynu.Markdown.Widget::links : MarkdownView -> Array LinkInfo` returning every link in the view's document in source-DFS order. This is a thin wrapper over `Yaynu.Markdown.Ast::collect_links(v.@document)` — same enumeration, same indices.

#### Scenario: Equivalence to collect_links

- **WHEN** `Widget::links(v)` is called for any `MarkdownView v`
- **THEN** the returned array SHALL equal `Ast::collect_links(v.@document)` element-for-element (same `index`, `url`, and `display_text` in the same order)

#### Scenario: Empty document yields empty array

- **WHEN** `Widget::links(MarkdownView::new(""))` is called
- **THEN** the returned array SHALL be empty

### Requirement: TAB navigation integration pattern

The system SHALL document the recommended way to drive TAB / Shift-TAB / Space navigation on top of `MarkdownView`. The pattern SHALL be: the caller maintains a `selected : I64` in their own application state, sets `RenderOptions.@selected_link = some(selected)` on the view's options before each render, and on Space looks up the URL via `Widget::links(view).@(selected).@url` (after bounds-checking). The library SHALL NOT consume TAB / Shift-TAB / Space itself in `MarkdownView::handle_key` — those keys remain available to the caller's key router.

#### Scenario: handle_key does not consume TAB

- **WHEN** `Widget::handle_key(Key::tab(), v)` is invoked
- **THEN** the returned view SHALL be structurally equal to `v` (TAB is left for the caller to handle)

#### Scenario: handle_key does not consume Shift-TAB

- **WHEN** any key the library treats as "unhandled" is passed to `handle_key` (including Shift-TAB if introduced upstream)
- **THEN** the view SHALL be returned unchanged

#### Scenario: handle_key Space-bar reservation

- **WHEN** `Widget::handle_key(Key::char(" "), v)` is invoked
- **THEN** the returned view's `@scroll` SHALL behave as documented in the existing `markdown-widget` capability (Space = PageDown by default)

> Note: the existing `markdown-widget` spec assigns Space to PageDown; this change does **not** modify that. Callers who want Space to mean "open link" intercept the key before passing it to `handle_key`, or filter it out of the keys they forward.
