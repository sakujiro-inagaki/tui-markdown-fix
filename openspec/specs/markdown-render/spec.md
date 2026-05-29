# markdown-render Specification

## Purpose

TBD - created by archiving change yaynu-markdown-view. Update Purpose after archive.
## Requirements
### Requirement: Public render functions

The system SHALL expose under `Yaynu.Markdown.Render`:
- `render : Document -> RenderOptions -> Rect -> Buffer -> (I64, Buffer)` — draws the document into the given `Rect` of the given `Buffer` and returns the number of rows used and the updated `Buffer`.
- `render_string : String -> RenderOptions -> Rect -> Buffer -> (I64, Buffer)` — equivalent to `Parser::parse` composed with `render`.
- `estimate_height : Document -> RenderOptions -> I64 -> I64` — given a target width, returns the number of rows the document would occupy if rendered without vertical clipping.

#### Scenario: Render returns row count

- **WHEN** `render` is called with a document whose layout occupies K rows in the given `Rect`
- **THEN** the returned `I64` SHALL equal K (or the row count actually written if K exceeds `rect.height`, in which case writing is clipped at `rect.height`)

#### Scenario: estimate_height matches render row count

- **WHEN** `estimate_height(doc, opts, width)` returns H
- **AND** `render(doc, opts, rect_of_width_and_tall_enough_height, buf)` returns `(R, _)`
- **THEN** H SHALL equal R

#### Scenario: render_string equals parse-then-render

- **WHEN** `render_string(src, opts, rect, buf)` returns `(R, B1)`
- **AND** `render(parse(src), opts, rect, buf)` returns `(R2, B2)`
- **THEN** R SHALL equal R2 and B1 SHALL equal B2

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

### Requirement: Paragraph rendering with word-aware wrapping

When `wrap = true`, paragraph rendering SHALL fill each row up to `rect.width` columns measured by `Tui.Width::string_width`, breaking at ASCII whitespace by preference and at codepoint boundaries as a fallback for over-wide tokens. Soft breaks between source lines SHALL render as a single space. Hard breaks (`line_break` inline) SHALL force a new row. A blank row SHALL separate consecutive blocks.

#### Scenario: Word wrap at whitespace

- **WHEN** a paragraph "alpha beta gamma" is rendered into a 9-column rect with `wrap = true`
- **THEN** the output SHALL break between words, producing rows whose visible content does not exceed 9 columns

#### Scenario: Over-wide token falls back to character break

- **WHEN** a single token is wider than `rect.width`
- **THEN** the renderer SHALL break it at codepoint boundaries rather than overflowing the row

#### Scenario: Hard break forces newline

- **WHEN** a paragraph contains a `line_break` inline between two text runs
- **THEN** the second run SHALL begin at column 0 of the next row, regardless of remaining width

#### Scenario: Trailing blank row

- **WHEN** two paragraph blocks are rendered consecutively
- **THEN** the output SHALL contain exactly one blank row between them

### Requirement: Heading rendering

Heading rendering SHALL apply the level-appropriate style from `MarkdownStyle.heading[level - 1]`, render the inline content on its own row (wrapping if needed), and follow the heading with one blank row.

#### Scenario: Level affects style index

- **WHEN** a level-3 heading is rendered
- **THEN** the renderer SHALL use the `MarkdownStyle.heading` entry at index 2

#### Scenario: Heading followed by blank row

- **WHEN** any heading is rendered and at least one block follows it
- **THEN** the heading row(s) SHALL be followed by exactly one blank row before the next block

### Requirement: Code-block rendering

Code-block rendering SHALL apply `MarkdownStyle.code_block` to the body. When `code_block_border = true`, the body SHALL be enclosed in a single-line top/bottom border (and optional side borders) drawn with light box-drawing glyphs, and the optional language label SHALL be embedded in the top border when `show_code_block_language = true`. When `wrap_code = false`, lines longer than the inner width SHALL be truncated and suffixed with `…`; when `wrap_code = true`, lines SHALL be hard-wrapped at codepoint boundaries.

#### Scenario: Bordered code block with language

- **WHEN** a code block with `language = Some("python")` is rendered with `code_block_border = true` and `show_code_block_language = true`
- **THEN** the first rendered row SHALL contain the literal substring `python` and a horizontal border glyph

#### Scenario: Unbordered code block

- **WHEN** the same code block is rendered with `code_block_border = false`
- **THEN** the output SHALL NOT contain box-drawing border glyphs, and the body lines SHALL be rendered with the code-block style only

#### Scenario: Long-line truncation

- **WHEN** a code-block line is longer than the available inner width and `wrap_code = false`
- **THEN** the rendered row SHALL end with `…` and SHALL NOT exceed `rect.width` columns

### Requirement: Blockquote rendering

Blockquote rendering SHALL prefix every row of its nested content with a single-column `│` marker styled by `MarkdownStyle.blockquote_marker`, indent the content by one column past the marker, and apply `MarkdownStyle.blockquote` to the body. Nested blockquotes SHALL stack additional markers.

#### Scenario: Single-level marker

- **WHEN** a single-level blockquote is rendered
- **THEN** every row produced by its content SHALL begin with the marker glyph `│`

#### Scenario: Nested marker stacking

- **WHEN** a blockquote containing another blockquote is rendered
- **THEN** rows of the inner content SHALL begin with two `│` markers, each at successive column positions

### Requirement: List rendering with nesting and task markers

List rendering SHALL indent items by `indent_unit * level` columns at nesting level `level` (zero-based), select the bullet glyph from `bullet_markers[level mod bullet_markers.length]` for unordered lists, and render ordered items as `N.` where N starts at `start` and increments by item index. Task items SHALL substitute `task_done_marker` (for `Some(true)`) or `task_pending_marker` (for `Some(false)`) for the bullet, styled by `MarkdownStyle.task_done` or `MarkdownStyle.task_pending` respectively.

#### Scenario: Three-item unordered list

- **WHEN** a flat unordered list with three items is rendered with `bullet_markers = ["•", "◦", "▪"]`
- **THEN** the three rendered item lines SHALL each begin with `•`

#### Scenario: Nested level cycles bullet marker

- **WHEN** a list contains a nested list at level 1, with `bullet_markers = ["•", "◦", "▪"]`
- **THEN** the inner items SHALL be rendered with `◦`

#### Scenario: Ordered list with start = 3

- **WHEN** an ordered list with `start = 3` and two items is rendered
- **THEN** the two item lines SHALL be prefixed with `3.` and `4.` respectively

#### Scenario: Task markers replace bullet

- **WHEN** a task list contains one done and one pending item
- **THEN** the rendered lines SHALL be prefixed with `task_done_marker` and `task_pending_marker` respectively

### Requirement: Table rendering with width allocation

Table rendering SHALL:
1. Compute the intrinsic visible width of each cell using `Tui.Width::string_width`.
2. Allocate each column its intrinsic max width if the sum (plus border overhead) fits within `rect.width`; otherwise scale columns proportionally with a minimum of 3 columns per cell.
3. Truncate any cell content that exceeds its allocated width by suffixing `…`.
4. Apply `MarkdownStyle.table_header` to header cells, `MarkdownStyle.table_cell` to body cells, and `MarkdownStyle.table_border` to border glyphs.
5. Honor `Alignment` per column (`left` / `center` / `right`; `none` is treated as `left`).
6. Use the glyph set selected by `RenderOptions.table_border` (`none`, `light`, `heavy`, `ascii`).

#### Scenario: Borderless table uses spaces between columns

- **WHEN** a table is rendered with `table_border = none`
- **THEN** the output SHALL contain no border glyphs and SHALL separate columns with spaces

#### Scenario: Light border draws box-drawing glyphs

- **WHEN** a table is rendered with `table_border = light`
- **THEN** the output SHALL contain at least one of `┌` `┐` `└` `┘` `─` `│` `┬` `┴` `┼` `├` `┤`

#### Scenario: Column truncation when too narrow

- **WHEN** a cell's content exceeds its allocated column width
- **THEN** the rendered cell SHALL end with `…` and SHALL fit exactly within its allocated width

#### Scenario: Right alignment pads on the left

- **WHEN** a column has `Alignment::right` and a cell with content narrower than the column
- **THEN** the rendered cell SHALL be padded with spaces on the left such that the visible content ends at the column's right edge

### Requirement: Thematic break rendering

The renderer SHALL render a `thematic_break` block as exactly one row consisting of `thematic_break_char` repeated to fill `rect.width` columns, styled by `MarkdownStyle.thematic_break`.

#### Scenario: Full-width rule

- **WHEN** a `thematic_break` is rendered in a 20-column rect with `thematic_break_char = "─"`
- **THEN** the rendered row SHALL consist of the character `─` repeated 20 times

### Requirement: Inline styles

The renderer SHALL apply:
- `MarkdownStyle.strong` to `strong` content.
- `MarkdownStyle.emph` to `emph` content.
- `MarkdownStyle.strikethrough` to `strikethrough` content.
- `MarkdownStyle.inline_code` to `code` inlines (rendered verbatim).
- `MarkdownStyle.link` to the display content of `link` inlines (the URL is not rendered in v0.1).
- `MarkdownStyle.image` to a synthesized text of the form `[image: <alt>]` for `image` inlines.
- The base text style to `raw_html` inlines (rendered verbatim).
- The base text style to `text` inlines.

Inline styles SHALL be composed with the surrounding block's base style by overlay (later styles override earlier styles' set attributes).

#### Scenario: Image becomes alt-text placeholder

- **WHEN** an `image` inline with alt `logo` is rendered
- **THEN** the visible output SHALL contain `[image: logo]`

#### Scenario: Link display content is styled and URL is not shown

- **WHEN** a `link` inline with display `Example` and URL `https://example.com` is rendered
- **THEN** the visible output SHALL contain `Example` and SHALL NOT contain `https://example.com`

### Requirement: East Asian Width and emoji correctness

The renderer SHALL measure every visible string with `Tui.Width::string_width` so that double-wide characters (e.g., CJK ideographs, full-width punctuation, common emoji) occupy two columns and do not cause column overflow or misalignment.

#### Scenario: Japanese paragraph respects rect width

- **WHEN** a paragraph containing Japanese text is rendered with `wrap = true`
- **THEN** every produced row SHALL have a visible width less than or equal to `rect.width`

#### Scenario: Mixed-width table column

- **WHEN** a table cell contains a mix of ASCII and CJK characters
- **THEN** its allocated column width SHALL be computed using `Tui.Width::string_width` so the right edge aligns

### Requirement: Clipping

When the document's rendered height exceeds `rect.height`, the renderer SHALL stop writing rows past row `rect.height - 1`, and SHALL still return a height value that reflects the rows actually written (bounded by `rect.height`).

#### Scenario: Document taller than rect

- **WHEN** a document whose `estimate_height` exceeds `rect.height` is rendered
- **THEN** no `Buffer` cells outside the `rect` SHALL be modified, and the returned row count SHALL equal `rect.height`
