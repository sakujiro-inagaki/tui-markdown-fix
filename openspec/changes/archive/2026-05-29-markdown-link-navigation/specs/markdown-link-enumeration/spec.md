## ADDED Requirements

### Requirement: LinkInfo data shape

The system SHALL provide a `LinkInfo` record carrying at least:
- `index : I64` — the link's source-DFS position in the document (0-based).
- `url : String` — the URL as captured by the parser, verbatim.
- `display_text : String` — the link's display content flattened to plain text, using the same projection that the table renderer uses for cell-width measurement (so `inlines_plain_text(kids)`).

#### Scenario: All three fields populated

- **WHEN** a `LinkInfo` is constructed for the link `[Example](https://example.com)`
- **THEN** `info.@index` SHALL be the link's source-DFS index, `info.@url` SHALL equal `"https://example.com"`, and `info.@display_text` SHALL equal `"Example"`

#### Scenario: Display text discards inline structure

- **WHEN** a `LinkInfo` is constructed for the link `[**bold** part](https://example.com)`
- **THEN** `info.@display_text` SHALL equal `"bold part"` (style markup discarded, internal whitespace preserved)

### Requirement: collect_links AST query

The system SHALL expose `Yaynu.Markdown.Ast::collect_links : Document -> Array LinkInfo` returning every `Inline::link` in the document, in source depth-first order, with `index` matching position in the returned array (`returned.@(i).@index == i`).

Traversal rules (so this enumeration matches the renderer's own counter):
- Visit `Document.@blocks` left-to-right.
- Within each block, visit inlines left-to-right.
- Recurse into container inlines (`emph`, `strong`, `strikethrough`, `link`) before advancing to the next sibling. A link nested inside another inline counts as a separate link.
- For container blocks (`blockquote`, `list` items, `table` headers / rows), recurse into the contained blocks / inlines using the same rules.

`image` inlines SHALL NOT contribute to the result.

#### Scenario: Empty document

- **WHEN** `collect_links(Document { blocks: [] })` is called
- **THEN** the returned array SHALL be empty

#### Scenario: Multiple links across blocks

- **WHEN** a document contains paragraph `[A](http://a)` followed by paragraph `[B](http://b)`
- **THEN** `collect_links` SHALL return two `LinkInfo`s with `index == 0` (URL `http://a`) and `index == 1` (URL `http://b`) in that order

#### Scenario: Nested-in-list link

- **WHEN** a document has a single bullet item whose content is `[X](http://x)`
- **THEN** `collect_links` SHALL contain one entry for `http://x` with `index == 0`

#### Scenario: Link in table cell

- **WHEN** a table row contains a cell whose only inline is `[T](http://t)`
- **THEN** `collect_links` SHALL include that link with the appropriate source-DFS index relative to all other links in the document

#### Scenario: Image is not a link

- **WHEN** a document contains the inline `![alt](http://img.example/x.png)`
- **THEN** `collect_links` SHALL NOT include it

#### Scenario: Index matches array position

- **WHEN** `links = collect_links(doc)` and `links.get_size > 0`
- **THEN** for every `i` in `[0, links.get_size)`, `links.@(i).@index` SHALL equal `i`

### Requirement: Index agreement with renderer

The `index` assigned by `collect_links` SHALL equal the link counter the renderer uses for `RenderOptions.selected_link`. Specifically: if a caller sets `selected_link = some(k)` and `links = collect_links(doc)`, the highlighted link SHALL be the one whose URL equals `links.@(k).@url` (assuming `k < links.get_size`).

#### Scenario: Selecting the n-th link by collect_links index

- **WHEN** a document has at least two links and the caller renders it with `selected_link = some(1)`
- **THEN** the highlighted link in the output SHALL be the same link whose `LinkInfo` is `collect_links(doc).@(1)`
