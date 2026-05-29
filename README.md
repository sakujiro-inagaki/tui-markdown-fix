# tui-markdown-fix

A Markdown rendering pipeline for the [Fix programming language](https://github.com/tttmmmyyyy/fixlang), targeted at terminal UIs built on [tui-fix](https://github.com/sakujiro-inagaki/tui-fix).

Parses a GitHub-Flavored Markdown subset into a typed AST, renders it into a `tui-fix` `Buffer`, and exposes a `MarkdownView` widget that drops into any `Frame`-based UI.

> **v0.1 namespace note.** The library currently lives under `Yaynu.Markdown.*` because it was extracted from a sibling project. The next major release will rename the namespace to `TuiMarkdown.*`. Plan accordingly when pinning.

## Scope

Designed to render the markdown LLMs actually emit — not to pass the CommonMark conformance suite.

### Supported

- Block: paragraphs, ATX headings (`#`…`######`), unordered & ordered lists, task lists (`[ ]`/`[x]`), nested lists, blockquotes (with nesting), fenced & indented code blocks, GFM tables, thematic breaks, HTML block passthrough.
- Inline: emphasis (`*` / `_`), strong (`**` / `__`), strikethrough (`~~`), inline code, links, images (rendered as `[image: alt]`), `http(s)://` autolinks, angle-bracket autolinks, hard breaks, raw HTML passthrough.
- Rendering: word wrapping with codepoint-boundary fallback, East Asian Width via `tui-fix`'s `Width::string_width`, configurable per-element styling (`MarkdownStyle`), four table border styles (`none` / `light` / `heavy` / `ascii`), optional bordered code blocks with embedded language label.

### Not supported (current limitations)

- Syntax highlighting inside code blocks.
- OSC 8 clickable hyperlinks — link enumeration + selected-link highlight (see [Link navigation](#link-navigation)) cover the navigation use case without it. May revisit if a real need surfaces.
- Image rendering — images become the literal placeholder `[image: <alt>]` (and are excluded from `Ast::collect_links`).
- Footnotes, math (`$...$`), mentions, emoji shortcodes, `<details>` folding, hard `<br>` outside of trailing-spaces convention.
- HTML parsing — both block- and inline-level raw HTML is passed through as text.
- Streaming / incremental parsing — each redraw re-parses the document. The widget caches the parsed `Document`, so this is only an issue if the source string changes every frame.
- CommonMark delimiter-run subtleties — inline matching is greedy outside-in. Adversarial inputs may render as literal text.
- Selected-link highlight inside table cells — table cells render as plain text in v0.2, so the highlight does not apply there. The link counter still advances correctly past links in cells, so subsequent links keep their right indices.

## Quick start

`fixproj.toml`:

```toml
[[dependencies]]
name = "tui-markdown"
version = "0.1.0"
git = { url = "https://github.com/sakujiro-inagaki/tui-markdown-fix.git" }
```

Render once to a buffer:

```fix
module Main;

import Yaynu.Markdown.Render;
import Yaynu.Markdown.Render.Options;
import Yaynu.Tui.Buffer;
import Yaynu.Tui.Rect;

main : IO ();
main = (
    let src = "# Hello\n\nWorld!";
    let opts = RenderOptions::default;
    let rect = Rect::make(0, 0, 40, 10);
    let buf = Buffer::empty(40, 10);
    let (_, _) = Render::render_string(src, opts, rect, buf);
    pure()
);
```

Use as a scrollable widget inside `Yaynu.Tui::run`:

```fix
import Yaynu.Markdown.Widget;
import Yaynu.Tui.Frame;
import Yaynu.Tui.Rect;

let view = MarkdownView::new(my_markdown_string);
// inside `view : s -> Frame -> Frame`:
//   frame.render_markdown_view(s.@view, body_rect)
// inside `update : Event -> s -> ...`:
//   match ev { key(k) => UpdateResult::continue $
//       s.mod_view(|v| handle_key(k, v)), _ => ... }
```

See [`examples/show.fix`](examples/show.fix) for a complete TUI pager (arrow keys, `j`/`k`, PageUp/Down, `g`/`G`, `q`/Esc, plus `TAB` / `Space` for link navigation when the document contains links — see [Link navigation](#link-navigation) below).

## Customizing styles

`MarkdownStyle` is a plain record. Three presets ship: `default`, `dark`, `monochrome`. Override any field with the auto-generated setter:

```fix
import Yaynu.Markdown.Style;
import Yaynu.Tui.Style;
import Yaynu.Term.Ansi;

let palette = MarkdownStyle::dark.set_link(
    Style::default.with_fg(Color::named(Color16::bright_magenta())).set_underline(true)
);
let opts = RenderOptions::default.set_style(palette);
```

## `RenderOptions` reference

| Field                       | Default                                  | Meaning                                                          |
| --------------------------- | ---------------------------------------- | ---------------------------------------------------------------- |
| `style`                     | `MarkdownStyle::default`                 | Per-element style palette.                                       |
| `wrap`                      | `true`                                   | Word-wrap paragraphs at the rect width.                          |
| `wrap_code`                 | `false`                                  | Character-wrap long code lines (`false` truncates with `…`).     |
| `indent_unit`               | `2`                                      | Columns per nesting level for lists.                             |
| `task_done_marker`          | `"[x]"`                                  | Glyph used for completed task items.                             |
| `task_pending_marker`       | `"[ ]"`                                  | Glyph used for pending task items.                               |
| `bullet_markers`            | `["•", "◦", "▪"]`                        | Rotated per nesting level (`level mod len`).                     |
| `code_block_border`         | `true`                                   | Draw a box around fenced code blocks.                            |
| `show_code_block_language`  | `true`                                   | Print the language string above / in the border of a code block. |
| `thematic_break_char`       | `"─"`                                    | Glyph repeated to fill the row for `---` / `***`.                |
| `table_border`              | `TableBorderStyle::light()`              | One of `none` / `light` / `heavy` / `ascii`.                     |
| `selected_link`             | `Option::none()`                         | Source-DFS index of the link to highlight; `none` = no highlight; out-of-range degrades to `none`. |
| `selected_link_style`       | `Style::default.set_reverse(true)`       | Style overlaid onto `MarkdownStyle.@link` for the selected link's cells. |

## Link navigation

The library exposes two primitives that together let a TUI implement
"TAB cycles links, Space opens the selected one":

- `Yaynu.Markdown.Ast::collect_links : Document -> Array LinkInfo` —
  enumerate every link in source-DFS order. Each `LinkInfo` carries
  `index : I64` (0-based, matches the array position),
  `url : String`, and `display_text : String` (the link's display
  content flattened via `inlines_plain_text`). Container inlines
  (`emph` / `strong` / `strikethrough` / `link`) recurse before
  advancing, so a link nested inside emphasis gets its own index.
  `image` inlines are excluded.
- `Yaynu.Markdown.Widget::links : MarkdownView -> Array LinkInfo` —
  the widget-side wrapper that returns
  `Ast::collect_links(v.@document)`. Use this if you've already wrapped
  the document in a `MarkdownView` and don't want to import the AST
  module just for enumeration.

To highlight the currently-selected link, set
`RenderOptions.@selected_link = some(idx)` (and customise
`selected_link_style` if the default reverse-video overlay doesn't
suit your palette). The renderer's internal link counter walks the
document in the same DFS order as `collect_links`, so `idx` always
refers to the same link in both APIs. An out-of-range `idx` is
rendered identically to `none` — no panic, no warning — so a stale
index after the source changes is harmless.

`MarkdownView::handle_key` deliberately does not consume `TAB`,
`Shift-TAB`, or `Space` for link navigation — those keys remain
available to your application's key router. The recommended pattern:

```fix
// In your application state, alongside the MarkdownView:
//   selected : I64,
//   links    : Array LinkInfo  -- precomputed via Widget::links

// In update, intercept Tab and Space before forwarding to handle_key:
tab() => (
    let n = s.@links.get_size;
    if n <= 0 { UpdateResult::continue(s) }
    else { UpdateResult::continue $ s.set_selected((s.@selected + 1) % n) }
),
char(c) => (
    if c == " " && s.@links.get_size > 0
        && s.@selected >= 0 && s.@selected < s.@links.get_size {
        // Open: use the URL from `s.@links.@(s.@selected).@url`.
        UpdateResult::continue(s)
    } else { UpdateResult::continue $ s.mod_view(|v| handle_key(k, v)) }
),

// In view, stamp the selected index onto the render options:
let opts = s.@view.@options.set_selected_link(
    if s.@links.get_size > 0 { Option::some(s.@selected) } else { Option::none() }
);
let view2 = s.@view.set_options(opts);
frame.render_markdown_view(view2, body_rect)
```

See [`examples/show.fix`](examples/show.fix) for a complete pager with
this wiring in place. (`Yaynu.Markdown` v0.2 deferred OSC 8 clickable
hyperlinks: the enumeration + highlight pair turns out to be enough
to ship the navigation UX entirely on the caller side without an
upstream `tui-fix` change.)

## Testing

```sh
fix test
```

The suite includes parser scenarios, render scenarios for each block type, a `MarkdownView` widget exercise, and snapshot fixtures under `tests/fixtures/snapshots/`. The snapshot harness auto-bootstraps a missing `.expected.txt` from the current renderer output — review the generated file before committing it.

## Building examples

Because `fix run -f` doesn't inherit the project's linker flags, examples need the C-shim flags on the command line:

```sh
fix run -f examples/all_features.fix \
    -s tui_shim -s term_shim \
    -L .fixlang/deps/tui_0.2.0/c_src \
    -L .fixlang/deps/term_0.2.0/c_src
```

(Adjust the version numbers in the paths to match your installed deps under `.fixlang/deps/`.)

## License

MIT.
