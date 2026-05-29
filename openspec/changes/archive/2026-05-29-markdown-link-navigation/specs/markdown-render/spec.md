## MODIFIED Requirements

### Requirement: RenderOptions and defaults

The system SHALL provide a `RenderOptions` record carrying at least: `style : MarkdownStyle`, `wrap : Bool`, `wrap_code : Bool`, `indent_unit : I64`, `task_done_marker : String`, `task_pending_marker : String`, `bullet_markers : Array String`, `code_block_border : Bool`, `show_code_block_language : Bool`, `thematic_break_char : String`, `table_border : TableBorderStyle`, `selected_link : Option I64`, and `selected_link_style : Style`. The system SHALL provide `RenderOptions::default` returning a value with sensible v0.1 defaults, including `selected_link = none()` and `selected_link_style = Style::default.set_reverse(true)`.

#### Scenario: Defaults are usable

- **WHEN** a caller uses `RenderOptions::default` with `MarkdownStyle::default`
- **THEN** rendering SHALL produce non-empty, non-corrupted output for any document the parser can produce

#### Scenario: TableBorderStyle constructors

- **WHEN** `RenderOptions::default.@table_border` is inspected
- **THEN** it SHALL be one of the `TableBorderStyle` constructors `none`, `light`, `heavy`, or `ascii`

#### Scenario: Default selected_link is none

- **WHEN** `RenderOptions::default.@selected_link` is inspected
- **THEN** it SHALL equal `Option::none()`

#### Scenario: Default selected_link_style is reverse video

- **WHEN** `RenderOptions::default.@selected_link_style` is inspected
- **THEN** it SHALL have `reverse = true` and other attributes equal to `Style::default`'s

## ADDED Requirements

### Requirement: Selected-link highlight

When `RenderOptions.@selected_link` is `Some(k)` and the document contains at least `k + 1` links in source-DFS order (as defined by the `markdown-link-enumeration` capability), the renderer SHALL overlay `selected_link_style` on top of the per-link `MarkdownStyle.@link` for every cell of that specific link's display content. All other links SHALL render with only `MarkdownStyle.@link`, as they would have without `selected_link`.

If `Some(k)` is out of range (`k < 0` or `k >= total_links_in_document`), the renderer SHALL render as if `selected_link = none()` — no highlight, no panic.

#### Scenario: Selected link gets the overlay

- **WHEN** a document contains two links `[A](http://a)` and `[B](http://b)` and is rendered with `selected_link = some(1)` and `selected_link_style = Style::default.set_reverse(true)`
- **THEN** the rendered cells of `B` SHALL carry `reverse = true` in addition to the regular link style, and the rendered cells of `A` SHALL NOT carry `reverse = true`

#### Scenario: None means no highlight

- **WHEN** the same document is rendered with `selected_link = none()`
- **THEN** every link cell SHALL render with `MarkdownStyle.@link` only — no `reverse` overlay anywhere

#### Scenario: Out-of-range index degrades to none

- **WHEN** a one-link document is rendered with `selected_link = some(5)`
- **THEN** the output SHALL be byte-identical to rendering with `selected_link = none()`

#### Scenario: Selected link spans multiple wrapped rows

- **WHEN** a link's display text wraps across two rows and that link is selected
- **THEN** every cell of the link's content on both rows SHALL carry the `selected_link_style` overlay

### Requirement: Link counter agrees with collect_links

The renderer's internal link counter SHALL increment in the same source-DFS order as `Yaynu.Markdown.Ast::collect_links`. For any document `doc` and any `k` such that `k < collect_links(doc).get_size`, rendering with `selected_link = some(k)` SHALL highlight the same link whose URL is `collect_links(doc).@(k).@url`.

#### Scenario: Index zero highlights the first link

- **WHEN** a document with at least one link is rendered with `selected_link = some(0)`
- **THEN** the highlighted link's URL SHALL equal `collect_links(doc).@(0).@url`

#### Scenario: Index one highlights the second link

- **WHEN** a document with at least two links is rendered with `selected_link = some(1)`
- **THEN** the highlighted link's URL SHALL equal `collect_links(doc).@(1).@url`
