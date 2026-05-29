## Context

`tui-markdown-fix` v0.1 renders links as a styled text span. The URL is dropped at tokenisation time and the renderer has no notion of "which link is currently selected" — so a TUI cannot drive a TAB-cycle / Space-to-open UI.

To support that pattern, two primitives are needed: link enumeration (so callers can iterate and address links by index) and a per-render "selected link" handle that the renderer can highlight. Constraints carried over from v0.1:

- `tui-fix` is still the only rendering primitive — no upstream changes in this revision (an earlier draft also added OSC 8 hyperlink support via `tui-fix`; that's been dropped from scope to keep this change self-contained).
- No `Minilib` — `Std` + `tui-fix` only.
- v0.1 callers must keep working without changes (additive only).
- The two-pass renderer architecture (tokenise → wrap) stays. New state is threaded through the existing entry points, not bolted on with parallel walkers.

## Goals / Non-Goals

**Goals:**
- Stable source-DFS link index that's the same in `collect_links` and in `RenderOptions.selected_link`. Callers can do `links.@(selected).@url` without re-deriving anything.
- Single render call still produces the right output — no "first call to learn positions, second call to render" pattern.
- Zero cost for callers who don't use either primitive. `RenderOptions::default.@selected_link = none()` means the renderer does no extra work beyond comparing against `none`.
- Snapshot tests continue to pass byte-for-byte (the existing snapshots dump cell `ch` only; nothing about the new highlight changes the on-screen text).

**Non-Goals:**
- OSC 8 clickable hyperlinks. (Deferred indefinitely. Would require a tui-fix upstream change to `Style` + `Diff` and we decided that wasn't worth the scope right now.)
- A general per-link styling callback `(I64, Style) -> Style`. `selected_link` covers the real use case. Revisit only if a multi-link-style need actually materialises.
- Image links — `image` inlines still render `[image: alt]` and are not enumerable as navigable links.
- Click event ingestion.
- Auto-scroll to bring the selected link on-screen — caller drives scroll explicitly.
- Streaming / partial re-render. Each redraw still goes through the full pipeline.

## Decisions

### D1. Link identification: source-DFS index, stable across passes

`collect_links` and the renderer's counter walk the document in identical depth-first source order:

1. For each `Block` in `Document.@blocks`, visit it.
2. Within a block, visit inlines left-to-right.
3. Recurse into container inlines (`emph`, `strong`, `strikethrough`, `link`) **before** advancing to the next sibling. (Yes: a link nested inside emphasis counts as a separate link with its own index. The display text comes from the `link`'s `kids`.)
4. For container blocks (`blockquote`, `list` items, `table` cells), recurse into the contained blocks / inlines using the same rules.

A pre-walk that assigns indices and a render-time counter both follow this contract. If they ever diverge, the snapshot tests will catch it (a `selected_link = 0` test asserts which link gets the highlight).

- **Alternative considered:** assign indices at parse time and store them on `Inline::link`. Rejected — would change the AST shape (breaking v0.1) and tie a render-time concern to the parser.
- **Alternative considered:** use a stable hash of the URL. Rejected — duplicate links would collide; URL is also user-visible noise to thread.

### D2. `RenderOptions.selected_link : Option I64` + `selected_link_style : Style`

A single index, not a set. The selected link's cells get `palette.@link` overlaid with `selected_link_style` (default `Style::default.set_reverse(true)`).

- **Why a single index, not a set?** The target UX is "one link at a time." A `Set I64` adds API surface for no real-world win.
- **Why a separate style field, not just a single `selected_link_style` that replaces?** Composability — the caller's selection highlight should layer over their normal link palette (typically link colour + underline), not erase it. The renderer uses the same `overlay` helper as the rest of the inline pipeline.
- **Default style:** `Style::default.set_reverse(true)`. Visible on any palette. Callers customise with `RenderOptions::default.set_selected_link_style(...)`.

### D3. Counter threading through the renderer

Every render function that handles inlines becomes counter-aware. Concretely the internal signatures grow a `start_idx : I64` parameter and a `next_idx : I64` return:

```
render_blocks       : Array Block -> RenderOptions -> I64 width -> I64 start_idx -> (Array RenderedLine, I64 next_idx)
render_block        : Block -> RenderOptions -> I64 width -> I64 start_idx -> (Array RenderedLine, I64 next_idx)
inlines_to_tokens   : Array Inline -> Style -> MarkdownStyle -> RenderOptions -> I64 start_idx -> (Array InlineToken, I64 next_idx)
render_table_lines  : Table -> RenderOptions -> I64 width -> I64 start_idx -> (Array RenderedLine, I64 next_idx)
```

The change is mechanical — every recursive call now passes the counter through. The public entry points `Render::render`, `Render::render_string`, `Render::estimate_height` and `MarkdownView::render` keep their existing signatures (they internally start at `0` and discard the final counter).

`inlines_to_tokens` takes `RenderOptions` (not just `MarkdownStyle`) because it now needs to read `selected_link` and `selected_link_style`. This is a minor API tightening for an internal-ish function — external callers don't typically reach for it.

- **Alternative considered:** pre-walk to build an `Array Inline → I64` map. Rejected — Fix has no identity-keyed map, and re-walking introduces a second traversal that must agree with the renderer's.
- **Alternative considered:** carry the counter in a process-wide cell (FFI). Rejected — turns a pure renderer into something not. Tests would have to synchronise.

### D4. `collect_links : Document -> Array LinkInfo`

A pure AST walk producing:

```
type LinkInfo = box struct {
    index        : I64,
    url          : String,
    display_text : String
};
```

`display_text` is the link's `kids` flattened with `Render.Inline::inlines_plain_text` — the same projection the table renderer uses for cell widths. Callers can render it themselves in a status bar without re-walking.

- **Where it lives:** `Yaynu.Markdown.Ast` (pure types and queries) plus a thin `Yaynu.Markdown.Widget::links` wrapper that calls it with the view's document.
  - Putting `LinkInfo` in `Ast.fix` does mean `Ast` gains an `import` of `Yaynu.Markdown.Render.Inline` for `inlines_plain_text`. To avoid the cycle (Render already imports Ast), `inlines_plain_text` is moved to a small new module `Yaynu.Markdown.Ast.Plain` (or inlined into `Ast.fix` since it's pure and ten lines). Decision: inline it into `Ast.fix` as `_inline_plain` / `inlines_plain_text`, then have `Render.Inline` re-export by importing `Ast` (which it already does).
- **Why `Array` not iterator?** Callers index into it by `selected`. A random-access Array is the right shape.
- **Empty document → empty array.** No special case.

### D5. `MarkdownView::links` over a dedicated public AST helper

The widget exposes `links : MarkdownView -> Array LinkInfo` so widget users don't have to import `Yaynu.Markdown.Ast` or know about `Document`. The underlying `Ast::collect_links` stays public because (a) it costs nothing to expose and (b) headless tools (no widget) can still enumerate links.

### D6. Out-of-range `selected_link` is silently a no-op

If `selected_link = some(k)` and `k < 0` or `k >= total_links_in_doc`, the renderer renders with no link highlighted (same as `none()`). No panic, no warning. Rationale: the index can become stale across re-renders (caller edits the source, link count drops) and the right behaviour is "no highlight" not "crash."

## Risks / Trade-offs

- **[Counter-threading invasive — touches every render function]** → Mitigation: keep the change mechanical (always thread `start_idx`, return `next_idx`). Public entry points keep their signatures. Snapshot tests catch any drift between `collect_links` and the renderer's counter.
- **[`selected_link` index out of range]** → Mitigation: renderer treats out-of-range as `none()` (see D6). No panic. Documented in spec.
- **[Counter walk and `collect_links` walk could drift]** → Mitigation: both go through the same DFS contract documented in D1 and exercised by a targeted test that asserts `collect_links` order matches the renderer's counter (render with `selected_link = some(i)` for each `i`, confirm the i-th link is the one highlighted).
- **[`display_text` for a link containing an image]** → Today `inlines_plain_text` expands `image` to `[image: alt]`. `LinkInfo.@display_text` will reflect that. Documented as expected.
- **[Inlining `inlines_plain_text` into `Ast.fix` slightly couples Ast to a rendering concern]** → It's a 10-line pure function with no rendering dependencies; pragmatic enough. If the dependency arrow ever feels wrong, extract to `Ast/Plain.fix`.

## Migration Plan

This change is additive — no caller breaks.

1. Move `inlines_plain_text` (and its `_inline_plain` helper) from `Render/Inline.fix` to `Ast.fix`. Update `Render/Inline.fix` to import the new home; `Render/Table.fix` and any callers also re-point. (Verifies via `fix check` before touching anything else.)
2. Add `LinkInfo` + `collect_links` to `Ast.fix`. Add `Widget::links` wrapper.
3. Add `selected_link` + `selected_link_style` to `RenderOptions`.
4. Thread the link counter through `inlines_to_tokens`, `render_block`, `render_blocks`, `render_table_lines`. Public entry points adapt internally (start at `0`, discard the final counter).
5. In `_inline_to_tokens` for `link((kids, url))`: when the link's source-DFS index equals `selected_link`, overlay `selected_link_style` onto the base style before recursing.
6. Re-run snapshot tests — only the `display_text` is in the buffer dump, so no snapshot fixtures should drift. Regenerate any that surprise us.
7. Update `examples/show.fix` to demonstrate TAB / Shift-TAB / Space (Space prints the URL to stderr; users wire their own opener if they want one).
8. README + CHANGELOG entry.

**Rollback:** every change is additive. If a regression surfaces, the new `selected_link` field can be ignored (it defaults to `none()`); `collect_links` and `Widget::links` have no other dependencies and can stay or be removed independently.

## Open Questions

- **Q1.** Should `examples/show.fix` actually shell out to `open(1)` on Space, or just print the URL? Lean towards print-only in the example to avoid platform-specific shelling-out; users wire their own opener.
- **Q2.** When `selected_link` points at a link that ends up *clipped* by the rect (off-screen due to scroll), should the renderer scroll-to-bring-into-view? **Out of scope for this change** — let the caller drive scroll explicitly. Revisit if "auto-scroll to selected" turns out to be a common pain.
