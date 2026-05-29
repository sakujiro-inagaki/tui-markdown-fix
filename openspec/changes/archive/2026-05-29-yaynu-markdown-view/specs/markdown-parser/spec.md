## ADDED Requirements

### Requirement: Public parse function

The system SHALL expose `Yaynu.Markdown.Parser::parse : String -> Document` as a pure total function that returns a `Document` for any input `String`, never raising.

#### Scenario: Empty input

- **WHEN** `parse("")` is called
- **THEN** the result SHALL be a `Document` with an empty `blocks` array

#### Scenario: Plain paragraph

- **WHEN** `parse("hello world")` is called
- **THEN** the result SHALL be a `Document` containing one `paragraph` block whose inlines render to the text `hello world`

### Requirement: ATX headings level 1–6

The parser SHALL recognize ATX headings of the form `# `, `## `, `### `, `#### `, `##### `, `###### ` at the start of a line (after optional 0–3 leading spaces) and produce a `heading` block with the corresponding level.

#### Scenario: Each heading level

- **WHEN** the input contains a single line beginning with N `#` characters followed by a space and text, for each N in 1..=6
- **THEN** the result SHALL contain one `heading` block whose level equals N and whose inline content is the trimmed text

#### Scenario: Seven hashes are not a heading

- **WHEN** the input is `####### too many`
- **THEN** the result SHALL contain a `paragraph` block (not a heading), because levels above 6 are not headings

### Requirement: Paragraph and soft-break handling

The parser SHALL group consecutive non-blank, non-special lines into a single `paragraph` block, joining their lines with a single space character in the resulting inline text (soft break = space). Blank lines SHALL terminate the paragraph.

#### Scenario: Multi-line paragraph collapses to one block

- **WHEN** the input is two consecutive non-blank lines `foo\nbar`
- **THEN** the result SHALL contain one `paragraph` block whose text content is `foo bar`

#### Scenario: Blank line separates paragraphs

- **WHEN** the input is `foo\n\nbar`
- **THEN** the result SHALL contain two `paragraph` blocks, in order

#### Scenario: Hard break via two trailing spaces

- **WHEN** a paragraph line ends with two or more spaces before the newline
- **THEN** a `line_break` inline SHALL be emitted at that position

### Requirement: Fenced code blocks

The parser SHALL recognize fenced code blocks opened and closed by a line consisting of three or more backticks (`` ``` ``) or three or more tildes (`~~~`), capturing the optional info string after the opening fence as the `language` and the lines in between (verbatim, no inline parsing) as the `content`.

#### Scenario: Backtick fence with language

- **WHEN** the input is ` ```python\nx = 1\n``` `
- **THEN** the result SHALL contain a `code_block` with `language = Some("python")` and `content = "x = 1"` (or `"x = 1\n"`; trailing newline is implementation-defined but consistent)

#### Scenario: Tilde fence without language

- **WHEN** the input opens with `~~~` and closes with `~~~`
- **THEN** the result SHALL contain a `code_block` with `language = None`

#### Scenario: Unclosed fence runs to end of input

- **WHEN** a fenced code block is opened but never closed before end-of-input
- **THEN** the parser SHALL close it implicitly at end-of-input and include all remaining lines as `content`

### Requirement: Indented code blocks

The parser SHALL recognize a run of lines each beginning with at least 4 spaces (or a tab) as a single `code_block` with `language = None`, preserving the lines verbatim minus the leading indent.

#### Scenario: Four-space indent

- **WHEN** the input is `    line1\n    line2`
- **THEN** the result SHALL contain a `code_block` whose `content` joins those lines (with the 4-space prefix stripped from each)

### Requirement: Blockquotes with nesting

The parser SHALL recognize a line beginning with `>` (optionally followed by a space) as a blockquote line and group consecutive blockquote lines into a `blockquote` block. Nested `>` markers (e.g., `>> nested`) SHALL produce nested blockquotes.

#### Scenario: Single-level blockquote

- **WHEN** the input is `> quoted`
- **THEN** the result SHALL contain one `blockquote` block whose nested block is a `paragraph` with text `quoted`

#### Scenario: Nested blockquote

- **WHEN** the input is `> outer\n>> inner`
- **THEN** the outer `blockquote` SHALL contain a `paragraph` followed by a nested `blockquote` containing a `paragraph` with text `inner`

### Requirement: Lists (unordered, ordered, task)

The parser SHALL recognize:
- Unordered list items beginning with `-`, `*`, or `+` followed by a space.
- Ordered list items beginning with a 1–9 digit run followed by `.` or `)` and a space.
- Task list items: an unordered or ordered marker immediately followed by `[ ]` or `[x]` / `[X]` and a space, which sets `task = Some(false)` or `Some(true)` respectively.

Consecutive items of the same kind SHALL form a single `MarkdownList`. Indentation SHALL produce nested lists or nested blocks inside a `ListItem`.

#### Scenario: Unordered list of three items

- **WHEN** the input is `- a\n- b\n- c`
- **THEN** the result SHALL contain one `MarkdownList` with `ordered = false`, three items, each holding a `paragraph` whose text is `a`/`b`/`c` respectively, and `task = None`

#### Scenario: Ordered list with explicit start

- **WHEN** the input is `3. first\n4. second`
- **THEN** the resulting `MarkdownList` SHALL have `ordered = true` and `start = 3`

#### Scenario: Task items

- **WHEN** the input is `- [x] done\n- [ ] todo`
- **THEN** the two items SHALL have `task = Some(true)` and `task = Some(false)` respectively

#### Scenario: Nested list via indentation

- **WHEN** an item line is followed by indented lines that themselves match a list pattern
- **THEN** those lines SHALL be parsed as a nested `list` block inside the parent item's `blocks` array

#### Scenario: Tight vs loose

- **WHEN** all items in a list are separated by zero blank lines
- **THEN** the resulting `MarkdownList` SHALL have `tight = true`; otherwise `tight = false`

### Requirement: Thematic break

The parser SHALL recognize a line consisting of three or more `-`, `*`, or `_` characters (with optional spaces between) as a `thematic_break` block.

#### Scenario: Dash rule

- **WHEN** the input line is `---`
- **THEN** the result SHALL contain a `thematic_break` block

#### Scenario: Mixed-character rule is not a break

- **WHEN** the input line is `-*-`
- **THEN** the result SHALL contain a `paragraph` block (not a `thematic_break`)

### Requirement: GFM tables

The parser SHALL recognize a GFM table as a header row of `|`-separated cells, followed immediately by a delimiter row whose cells consist of `-` runs optionally bracketed by `:` to encode alignment, followed by zero or more data rows. The result SHALL be a `table` block.

#### Scenario: Simple two-column table

- **WHEN** the input is:
  ```
  | a | b |
  | - | - |
  | 1 | 2 |
  ```
- **THEN** the result SHALL contain one `table` block with two header cells (`a`, `b`), two `alignments` both `none`, and one row `[1, 2]`

#### Scenario: Alignment from colons

- **WHEN** a delimiter row contains `| :--- | :---: | ---: |`
- **THEN** the `alignments` array SHALL be `[left, center, right]`

#### Scenario: Missing delimiter row degrades to paragraph

- **WHEN** a line that looks like a table header is followed by a non-delimiter line
- **THEN** the result SHALL contain a `paragraph` block, not a `table`

### Requirement: Inline emphasis, strong, and strikethrough

The parser SHALL recognize:
- `*x*` and `_x_` as `emph` wrapping `x`.
- `**x**` and `__x__` as `strong` wrapping `x`.
- `~~x~~` as `strikethrough` wrapping `x`.

Unmatched delimiters SHALL be left as literal text. Inside-word `_` (e.g., `foo_bar_baz`) SHALL NOT produce emphasis (GFM behavior).

#### Scenario: Bold and italic

- **WHEN** the input is `**bold** and *italic*`
- **THEN** the paragraph's inline content SHALL contain a `strong` wrapping `bold`, the text ` and `, and an `emph` wrapping `italic`

#### Scenario: Strikethrough

- **WHEN** the input is `~~gone~~`
- **THEN** the paragraph's inline content SHALL contain a `strikethrough` wrapping `gone`

#### Scenario: Inside-word underscore is literal

- **WHEN** the input is `foo_bar_baz`
- **THEN** the paragraph's inline content SHALL be a single `text` of `foo_bar_baz` with no `emph`

### Requirement: Inline code

The parser SHALL recognize a `` ` ``-delimited run as an `inline_code` value carrying the text between the backticks verbatim (no further inline parsing inside).

#### Scenario: Single-backtick code

- **WHEN** the input is `` `x = 1` ``
- **THEN** the inline content SHALL be a `code` carrying the string `x = 1`

#### Scenario: Unclosed backtick is literal

- **WHEN** the input is `` `unclosed ``
- **THEN** the inline content SHALL contain a literal text starting with a backtick (no `code` constructor)

### Requirement: Links, images, and autolinks

The parser SHALL recognize:
- `[text](url)` as a `link` with `Array Inline` content (recursively parsed) and the `url` string.
- `![alt](url)` as an `image` with the `alt` string and the `url` string.
- `<url>` where `url` looks like a URL, and bare `http(s)://...` runs in text, as `link` values whose display content is a single `text` equal to the URL (GFM autolink behavior).

#### Scenario: Link

- **WHEN** the input is `[Example](https://example.com)`
- **THEN** the inline content SHALL contain a `link` whose display text is `Example` and whose URL is `https://example.com`

#### Scenario: Image

- **WHEN** the input is `![logo](https://example.com/x.png)`
- **THEN** the inline content SHALL contain an `image` with alt `logo` and URL `https://example.com/x.png`

#### Scenario: GFM autolink in text

- **WHEN** a paragraph contains the bare text `see https://example.com for more`
- **THEN** the inline content SHALL contain a `link` to `https://example.com`

### Requirement: Raw HTML passthrough

The parser SHALL NOT parse HTML in v0.1. Lines that begin a recognized HTML block (e.g., `<div>`) SHALL be captured as `html_block` carrying the raw text; HTML tags inside inline content SHALL be captured as `raw_html` inlines carrying the tag text verbatim.

#### Scenario: HTML block

- **WHEN** the input begins with `<div>content</div>` on its own paragraph
- **THEN** the result SHALL contain an `html_block` whose payload is that raw text

#### Scenario: Inline HTML tag

- **WHEN** the input is `a <span>b</span> c`
- **THEN** the paragraph's inline content SHALL include `raw_html` entries carrying `<span>` and `</span>` literally

### Requirement: Totality and no panics

The parser SHALL terminate on any input and SHALL NOT panic, abort, or raise. Malformed markdown SHALL degrade to a best-effort `Document` (typically a single `paragraph` containing the literal text).

#### Scenario: Random byte input

- **WHEN** `parse` is invoked on a non-empty arbitrary `String`
- **THEN** it SHALL return a `Document` value within finite time, without raising
