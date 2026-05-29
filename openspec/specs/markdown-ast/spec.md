# markdown-ast Specification

## Purpose

TBD - created by archiving change yaynu-markdown-view. Update Purpose after archive.
## Requirements
### Requirement: Document type

The system SHALL provide a `Document` value type representing a parsed markdown source as an ordered sequence of `Block`s.

#### Scenario: Empty document

- **WHEN** a `Document` is constructed from an empty input
- **THEN** its `blocks` array SHALL be empty (length zero)

#### Scenario: Block ordering preserved

- **WHEN** a `Document` is constructed from a source containing multiple top-level blocks
- **THEN** the order of entries in its `blocks` array SHALL match the order of the corresponding source elements top-to-bottom

### Requirement: Block union covers v0.1 GFM block elements

The system SHALL provide a `Block` tagged union with exactly the constructors required by v0.1: `paragraph`, `heading` (carrying a level 1–6 and its inline content), `code_block`, `blockquote` (carrying nested blocks), `list`, `table`, `thematic_break`, and `html_block`.

#### Scenario: Heading carries level and inlines

- **WHEN** a heading block is constructed
- **THEN** its payload SHALL be a tuple of an `I64` level in the inclusive range `[1, 6]` and an `Array Inline` content

#### Scenario: Blockquote is recursive

- **WHEN** a blockquote contains nested block content (including another blockquote)
- **THEN** the `blockquote` constructor SHALL hold an `Array Block` allowing arbitrary nesting

### Requirement: CodeBlock carries optional language and raw content

The system SHALL represent code blocks with a `CodeBlock` record having a `language : Option String` and a `content : String` that may contain newline characters.

#### Scenario: Fenced code block with language

- **WHEN** a fenced code block is parsed with an info string
- **THEN** its `language` SHALL be `Some(info)` and its `content` SHALL preserve internal newlines as written

#### Scenario: Indented code block without language

- **WHEN** an indented (4-space) code block is parsed
- **THEN** its `language` SHALL be `None`

### Requirement: MarkdownList and ListItem encode ordering, tightness, and task state

The system SHALL provide a `MarkdownList` record with `ordered : Bool`, `start : I64`, `tight : Bool`, and `items : Array ListItem`, and a `ListItem` record with `task : Option Bool` and `blocks : Array Block`.

#### Scenario: Unordered task list item

- **WHEN** an item is parsed from `- [x] done`
- **THEN** its `task` SHALL be `Some(true)` and its `blocks` SHALL begin with a `paragraph` whose inline content is the text `done`

#### Scenario: Pending task list item

- **WHEN** an item is parsed from `- [ ] todo`
- **THEN** its `task` SHALL be `Some(false)`

#### Scenario: Non-task item

- **WHEN** an item is parsed from `- plain bullet`
- **THEN** its `task` SHALL be `None`

#### Scenario: Ordered list start preserved

- **WHEN** an ordered list begins at `3.`
- **THEN** the resulting `MarkdownList` SHALL have `ordered = true` and `start = 3`

### Requirement: Table records headers, alignments, and rows

The system SHALL provide a `Table` record with `headers : Array (Array Inline)`, `alignments : Array Alignment`, and `rows : Array (Array (Array Inline))`, and an `Alignment` union with constructors `left`, `center`, `right`, and `none`.

#### Scenario: Alignment row parses left/center/right/none

- **WHEN** a delimiter row contains `| :--- | :---: | ---: | --- |`
- **THEN** the resulting `alignments` array SHALL be `[left, center, right, none]` in that order

#### Scenario: Row count and column count

- **WHEN** a table has N header columns and M body rows
- **THEN** `headers.length` SHALL equal N, `alignments.length` SHALL equal N, and `rows.length` SHALL equal M with each inner array having length N

### Requirement: Inline union covers v0.1 GFM inline elements

The system SHALL provide an `Inline` tagged union with exactly the constructors required by v0.1: `text`, `emph`, `strong`, `strikethrough`, `code`, `link` (carrying inline content and a URL string), `image` (carrying alt text and a URL string), `line_break`, and `raw_html`.

#### Scenario: Nested emphasis

- **WHEN** an inline `strong` constructor wraps inline children that include an `emph`
- **THEN** the AST SHALL faithfully reflect that nesting without flattening

#### Scenario: Link payload shape

- **WHEN** a `link` inline is constructed
- **THEN** its payload SHALL be a tuple of `Array Inline` (display content) and `String` (URL)

#### Scenario: Image payload shape

- **WHEN** an `image` inline is constructed
- **THEN** its payload SHALL be a tuple of `String` (alt text) and `String` (URL)
