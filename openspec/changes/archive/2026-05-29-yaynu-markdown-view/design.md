## Context

Yaynu is a TUI built on `tui-fix` whose dominant content type is LLM responses. Those responses arrive as GitHub Flavored Markdown — headings, bullet lists, fenced code, tables, inline emphasis — and are currently rendered with `Tui.Widget.Paragraph`, which treats everything as plain wrapped text. The result is visually flat: code blocks lose their structure, headings don't stand out, tables collapse into broken pipe-separated text.

Constraints we are working within:
- **`tui-fix` is the only external rendering primitive.** `Buffer` / `Style` / `Width` handle the terminal model, including East Asian Width. No new TUI engine.
- **No Minilib.** Yaynu and tui-fix both avoid Minilib; this change continues that policy. Only `Std` + `tui-fix`.
- **Fix language ergonomics.** Pure functions over `String` and array-of-records ASTs; no global mutation. Reference-counted in-place updates are acceptable but not architecturally required for v0.1.
- **Future extraction.** This is morally `tui-markdown-fix`, temporarily living under `Yaynu.Markdown.*`. We want the v0.2 extraction to be a namespace rename, not a redesign.
- **LLM-output bias over CommonMark purity.** We optimize for "the things LLMs actually emit," not spec compliance. Edge cases of inline emphasis matching, lazy continuation, etc. are not worth implementing in v0.1.

## Goals / Non-Goals

**Goals:**
- Render a representative LLM response (headings + lists + a code block + a table + emphasis + a link) legibly inside a `tui-fix` `Rect`, including Japanese / CJK content, without column breakage.
- Provide a clean three-layer architecture (`Ast` data / `Parser` pure function / `Render` Buffer writer) so each layer is independently testable.
- Provide `MarkdownView`, a drop-in for `Paragraph` with scroll support, so the Yaynu integration is a one-line swap at the call site.
- Keep the public API (`Yaynu.Markdown` facade, `RenderOptions`, `MarkdownStyle`, `MarkdownView`) stable enough that v0.2 (syntax highlighting, OSC 8 links) is additive — no breaking changes on existing fields.
- Snapshot-test the renderer using a textual `Buffer::debug_string` so visual regressions are caught in `fix test`.

**Non-Goals:**
- Full CommonMark conformance. We do not aim to pass the CommonMark spec suite. We aim to handle LLM output well.
- Syntax highlighting (deferred to v0.2 via injection point).
- Clickable / OSC 8 links (deferred).
- Footnotes, math (`$...$`), mentions, emoji shortcodes, `<details>`, rich `<br>` — all explicitly out for v0.1.
- HTML parsing. Raw HTML blocks/inlines are passed through as text.
- Streaming / incremental parse. Each redraw re-parses the whole document. (Memoization is an optimization we may add inside `MarkdownView` if needed, but the parser API stays pure.)
- Image rendering. Images become `[image: alt]` text only.

## Decisions

### D1. AST shape: tagged unions over open extension

We model `Block` and `Inline` as `box union`s with one constructor per element type, matching the spec sketch in the proposal. Renderer is a `match` over those constructors.

- **Alternative considered:** a generic `Node { kind: String, children: ..., attrs: ... }` (more extensible, parser-agnostic).
- **Why rejected:** loses the type checker as a renderer-completeness check. With tagged unions, adding a new block type forces every renderer site to be updated. For a small fixed set of GFM constructs, that's the property we want.
- **Extension path:** when v0.2 adds e.g. `math`, it's a new constructor. Renderers must update — that's a deliberate compile-time signal, not a regression.

### D2. Parser: two passes, line-based block + state-machine inline

`Parser.parse` runs two passes:

1. **Block pass** — split input by `\n`, iterate lines, maintain a stack of "open containers" (blockquote, list, list-item) and the current "leaf" (paragraph, fenced code, indented code). Close containers on dedent or terminator lines. Output: `Array Block` with raw inline strings still attached.
2. **Inline pass** — for each leaf that holds inline text, run a small state machine over the bytes: track `*`/`_`/`~~`/`` ` ``/`[...](...)`/`![...](...)` delimiters and emit `Array Inline`. Unmatched delimiters degrade to literal text.

- **Alternative considered:** a single-pass scanner.
- **Why rejected:** GFM blocks need lookahead (table detection needs to see the separator line; fenced code is delimited by a sentinel); inline parsing inside a paragraph has zero context from outside. Two passes keep each loop simple.
- **"Naive" inline parsing is a stated v0.1 limitation.** Specifically we punt on CommonMark's left/right-flanking delimiter run rules — runs of `*`/`**` are matched greedily from outside in, single-pass. We document the limitation; in practice, LLM output is well-behaved.

### D3. Rendering model: write directly into `Buffer`, return `(used_height, Buffer)`

The renderer is a downward recursion that takes an `(indent, base_style, rect, buf)` context and returns `(rows_written, buf)`. No intermediate "rendered line" representation.

- **Alternative considered:** produce an `Array (StyledLine)` intermediate, then blit to `Buffer`.
- **Why we still chose direct:** keeps memory profile simple (no second materialization), and aligns with how `tui-fix`'s `Paragraph` already does layout. The intermediate layer would be helpful for an upcoming "estimate then render" pattern, but we get that today via a separate `estimate_height` pass that mirrors the structure of `render` without writing to the Buffer. If estimate/render drift becomes a real bug source, we revisit.

### D4. Wrapping: token-stream wrapper with word/word-fragment fallback

Inline rendering for paragraphs:

1. Flatten the `Array Inline` into a stream of `(text_fragment, style)` tokens, breaking text fragments at ASCII whitespace.
2. Greedy line fill: accumulate tokens until adding the next token would exceed `rect.width` (measured via `Tui.Width::string_width`). Emit a line; start the next.
3. If a single token is wider than `rect.width`, fall back to character-by-character splitting at codepoint boundaries.
4. Soft line breaks (newlines inside a paragraph) → single space. Hard breaks (`line_break` inline) → forced line.

For code blocks: by default `wrap_code = false`; long lines are truncated at `rect.width - 1` and suffixed with `…`. The user can flip `wrap_code = true` to character-wrap.

### D5. Tables: two-pass width allocation

1. Measure intrinsic width of every cell (`string_width` of the rendered inline plaintext — styling doesn't affect width).
2. Take the per-column max; sum.
3. If sum + border overhead ≤ `rect.width`: use intrinsic widths.
4. Else: scale columns proportionally, with a minimum of 3 chars per column; cells that overflow truncate with `…`.
5. Render header + separator + body using `TableBorderStyle` glyphs (`light` / `heavy` / `ascii` / `none`).

The truncate-with-`…` rule is the same as code blocks; consistency over cleverness.

### D6. Style injection: a record, not a typeclass

`MarkdownStyle` is a plain record. We expose `default`, `dark`, `monochrome` constructors. Users override individual fields with `.set_*` helpers.

- **Alternative considered:** a typeclass `Theme` with methods.
- **Why rejected:** themes are data, not behavior. Records are easier to compose, copy, and override field-by-field; they also serialize naturally if Yaynu later wants user-configurable themes.

### D7. Widget: `MarkdownView` owns `(document, options, scroll)`

`MarkdownView` is a record holding the parsed `Document`, the `RenderOptions`, and a scroll offset. `new` parses once. `with_scroll` / `with_options` produce updated copies. `render` calls the renderer with a `rect` adjusted by the scroll offset. `handle_key` mutates only `scroll`, clamped to `[0, total_height(width) - rect.height]`.

- **Why store the parsed `Document`?** Re-parsing every frame is wasteful, and the source `String` is typically immutable for the lifetime of the view (Yaynu sets it when a response completes).
- **What about streaming responses?** Out of scope for v0.1. When Yaynu wants to show tokens as they stream, it can re-create the `MarkdownView` per chunk. If that becomes hot, we add an `append` API in v0.2 — not now.

### D8. Test surface: `Buffer::debug_string` + snapshot fixtures

We add a `Yaynu.Markdown.Test` helper that converts a `Buffer` to a plain `String` (one row per line, styles discarded). Snapshot fixtures pair `<name>.md` with `<name>.expected.txt`. Diff on mismatch.

- **Alternative considered:** assert per-cell styles directly.
- **Why rejected:** style assertions are brittle (any palette tweak breaks them) and miss layout bugs. Layout is where the real bugs live; the textual snapshot catches layout regressions cheaply. Inline style behavior is covered by a small number of targeted assertions in the renderer unit tests, not by snapshots.

### D9. Namespace under `Yaynu.Markdown.*`, not `TuiMarkdown.*`

The library lives at `Yaynu.Markdown` for v0.1. README documents the planned extraction. We avoid creating a second project (`tui-markdown-fix`) prematurely — extraction is cheaper after the API has been validated against real LLM output in Yaynu.

## Risks / Trade-offs

- **[Naive inline parsing produces wrong output for adversarial markdown]** → Mitigation: documented limitation; snapshot fixtures cover the patterns LLMs actually emit. If a user-reported case slips through, add it as a regression fixture, not as a parser overhaul.
- **[Re-parse-on-redraw is O(N) per frame]** → Mitigation: `MarkdownView` caches the parsed `Document`. The render itself is the per-frame cost, which is bounded by `rect.height` since we stop writing past the viewport.
- **[East Asian Width edge cases (emoji ZWJ sequences, regional indicators) may still miscount]** → Mitigation: rely on `tui-fix`'s width logic; treat any defect as a `tui-fix` bug, not a markdown bug. Snapshot tests with Japanese / emoji catch the common cases.
- **[Table proportional scaling can produce ugly narrow columns]** → Mitigation: 3-char minimum per column; truncate with `…`. If a column is too narrow, we degrade visibly rather than overflow silently.
- **[Estimate-vs-render drift]** → Mitigation: both code paths share helper functions for line-counting (e.g. paragraph wrapping returns a count function used by `estimate_height`). Snapshot tests indirectly catch drift because scrolling depends on `total_height` being right.
- **[Namespace rename later is a breaking change for callers]** → Mitigation: only Yaynu calls this code in v0.1; we own all call sites. README states the rename plan up front.
- **[No streaming render]** → Accepted limitation for v0.1. Yaynu can rebuild the view on each token; if profiling shows pain we revisit.

## Migration Plan

This change is additive — no existing Yaynu feature is removed. The cutover is:

1. Land `Yaynu.Markdown.*` (parser + render + widget + tests) with `MarkdownView` unused.
2. Switch the LLM-response panel from `Paragraph` to `MarkdownView` in Yaynu's view code. Keep the change isolated to that one call site.
3. Verify manually with `examples/llm_response.fix` and a real Yaynu session.
4. **Rollback:** the panel swap is one commit; revert that single commit to fall back to `Paragraph`. The `Yaynu.Markdown.*` code can stay in tree unused, since it has no runtime cost when not constructed.

No data migration. No persisted format. No external API touched.

## Open Questions

- **Q1.** Should `MarkdownView` cache `total_height(width)` per-width? It's used twice per redraw (clamp + scrollbar). Likely fine to compute twice for v0.1; revisit if profiling complains.
- **Q2.** Do we want a `MarkdownStyle::yaynu` preset matched to Yaynu's existing color scheme, separate from `default`/`dark`/`monochrome`? Decide during the integration step — not needed to land the library.
- **Q3.** When the table can't fit even at 3-char minimums, do we (a) horizontal-scroll, (b) drop columns, or (c) overflow? v0.1 picks (c) overflow + truncate the rightmost columns. Revisit if LLM output regularly produces wide tables.
