## ADDED Requirements

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
