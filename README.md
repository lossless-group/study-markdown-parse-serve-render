# Markdown — Parse, Serve, Render — Study

## The question

> How do mature projects **parse, serve, and render Markdown** —
> across CommonMark, GitHub-Flavored, Pandoc-extended (Quarto / R Markdown),
> and directive / wikilink-flavored dialects — and which conventions are
> converging at each of the three stages?

The interesting bits are not the prose-to-HTML transform; that's commodity.
The interesting bits are: how does each system represent the *parsed* form
(AST, CST, lossless tree, token stream)? What does the *served* form look
like over the wire — JSON AST, HTML, MDX, raw + sidecar, language-server
protocol? And how is the *rendered* form composed — what extension points
exist, what runs at build time vs. request time vs. in-editor?

This study is the foundation for the [[lossless-flavored-markdown]] (LFM)
work and the broader [[astro-knots]] family — every Astro Knots site has
to make these choices, and reinventing them per-site is how we end up
with three subtly different definition-list renderings across the tree.

## What we are looking at, repo by repo

When reading each entry below, the working checklist is:

- **Dialect surface.** CommonMark only? GFM? Pandoc extensions (fenced
  divs, grid tables, definition lists, citations, inline footnotes)?
  Custom directives? Wikilinks?
- **Parser shape.** AST or CST? Lossless (round-trippable to source) or
  lossy? Token-stream or tree? What's the public API the caller sees?
- **Extension model.** How do you add a directive, a component, a custom
  block type? Plugin architecture, visitor pattern, or fork-and-patch?
- **Embedded code handling.** Fenced code blocks — do they get treated
  as opaque payload, syntax-highlighted, executed (Quarto), or routed
  to an external formatter?
- **Serve surface.** What does this thing expose to other tools? CLI?
  Library? Language Server Protocol? Wire format?
- **Render targets.** HTML? React/JSX? Astro components? PDF via LaTeX?
  Mermaid / KaTeX subprocesses?
- **Operational story.** Build-time? Runtime? Editor-time (LSP)? All
  three?
- **Frontmatter.** YAML, TOML, JSON? First-class or treated as opaque?

---

## The design space at a glance

| Stage | Bet | Entry |
|---|---|---|
| Parse + Serve (LSP) + Lint + Format | Pandoc-extended Markdown as a first-class CST with LSP delivery | [panache](./panache) |
| Parse + Convert — the upstream dialect spec | The universal markup converter; defines the Pandoc Markdown dialect other tools track (or fail to track) | [pandoc](./pandoc) |
| Parse — JS AST + plugin ecosystem | Markdown as a typed mdast tree; everything is a plugin over the unified AST | [remark](./remark) |
| Parse — Rust AST (mdast-compatible) | The same AST design as remark, ported to Rust by the same author | [markdown-rs](./markdown-rs) |
| Author + Render — JSX-free tag discipline | `{% tag %}` instead of MDX's JSX; schema-validated authoring for docs sites | [markdoc](./markdoc) |
| Serve (LSP) — wikilink / Zettelkasten focus | Markdown LSP centered on wiki-link cross-references and note-graphs, not Pandoc extensions | [marksman](./marksman) |

---

## In the study now

### [panache](./panache)
- **Repo:** https://github.com/jolars/panache — *Language server, formatter,
  and linter for Markdown, Quarto, and R Markdown*
- **Maintainer:** Johan Larsson (`jolars`)
- **Why this is here:** The cleanest current example of treating
  Pandoc-flavored Markdown — including Quarto `.qmd` and R Markdown `.Rmd`
  — as a **first-class dialect** rather than a lossy approximation on top
  of CommonMark. Built in Rust around a **lossless CST parser** (so the
  source can be round-tripped, formatted, and partially edited without
  flattening extensions), and delivered through three surfaces from one
  binary: an **LSP** (editor diagnostics, format-on-save, code actions),
  a **CLI formatter** (pre-commit, CI), and a **linter** (style +
  correctness rules). Handles the Pandoc extensions other tools quietly
  flatten — fenced divs, grid tables, definition lists, citations
  (`@key`), inline footnotes. Includes a notable pluggable design move:
  embedded code blocks (R chunks, Python chunks, SQL) can be routed to
  **external formatters** (`styler`, `black`, `sqlfluff`) — the LSP
  orchestrates rather than reinventing every language's formatter.
  Already ships extensions for VS Code (Marketplace + Open VSX). MIT
  licensed. Announced upstream in
  [pandoc#11544](https://github.com/jgm/pandoc/discussions/11544) and on
  [Posit Community](https://forum.posit.co/t/announcing-panache-a-language-server-formatter-and-linter-for-quarto-pandoc-and-r-markdown/210706).
  The interesting question against future entries: panache's CST design
  is the **lossless-tree bet** — compare it against AST-based parsers
  (remark, markdown-rs) that work on a lossy `mdast` tree.

### [remark](./remark)
- **Repo:** https://github.com/remarkjs/remark — *markdown processor
  powered by plugins, part of the @unifiedjs collective*
- **Maintainer:** Titus Wormer (`wooorm`) and the unified collective
- **Why this is here:** The dominant JS Markdown processor and the one
  the entire Lossless toolchain already depends on transitively — Astro's
  content pipeline, MDX, Markdoc's competitors, and most static-site
  generators in the JS world build on remark. It is the canonical
  **AST bet**: parse to `mdast` (a typed JSON tree of node objects),
  transform via plugins that walk and mutate the tree, then serialize.
  100% CommonMark, 100% GFM with a plugin, 100% MDX with a plugin —
  the dialect coverage that makes it the de-facto JS standard. The
  150+-plugin ecosystem is itself a design exhibit: extension via
  visitor-style tree walks rather than parser forks. Built on
  [`micromark`](https://github.com/micromark/micromark) (the
  specification-tracking tokenizer below remark), which is worth a
  separate look later. The interesting comparison against panache:
  remark's `mdast` is lossy by design — it does not promise
  source-round-trip — while panache's CST does. For our [[lossless-flavored-markdown]]
  work this is the upstream we extend, not a competitor.

### [markdown-rs](./markdown-rs)
- **Repo:** https://github.com/wooorm/markdown-rs — *CommonMark compliant
  markdown parser in Rust with ASTs and extensions*
- **Maintainer:** Titus Wormer (`wooorm`) — same author as remark and
  micromark
- **Why this is here:** The **Rust port of remark's design** by the
  remark author himself. 100% CommonMark, 100% GFM, 100% MDX, plus
  frontmatter and math extensions. 100% safe Rust, 100% safe HTML by
  default, 2300+ tests with 100% coverage and fuzz testing — the
  industrial-strength claim. Emits the same `mdast` shape as remark, so
  a tool can in principle share AST consumers across the JS and Rust
  worlds. The interesting comparisons: against **panache** (also Rust,
  but CST-based and Pandoc-flavored where markdown-rs is AST-based and
  CommonMark/GFM/MDX-focused) and against **pulldown-cmark / comrak**
  (the other major Rust parsers — different lineage, different API
  surface). For the "if we ever want a Rust-side Lossless parser"
  question this is the most direct continuation of the remark tradition.

### [markdoc](./markdoc)
- **Repo:** https://github.com/markdoc/markdoc — *A powerful, flexible,
  Markdown-based authoring framework*
- **Maintainer:** Stripe (`markdoc` org) — designed to power
  [Stripe's public docs](https://stripe.com/docs)
- **Why this is here:** The most credible **JSX-free alternative to MDX**
  in the docs-site space. Where MDX says "Markdown + JSX, components
  imported directly into prose," Markdoc says "Markdown + a constrained
  `{% tag %}` syntax with schema validation at the boundary." The
  tradeoff is explicit: Markdoc gives up MDX's component-import
  flexibility in exchange for (1) authoring that doesn't require
  authors to know JS/React, (2) schema-validated tags that fail loudly
  rather than silently rendering wrong, (3) clean separation between
  content and runtime components. Stripe's docs are the existence proof
  for the design at scale. For the render-stage comparison this is the
  third pole of the triangle — MDX (JSX-native), Astro Content (remark
  + components via slots), and Markdoc (tag-and-schema) — and the
  question worth asking is which pole [[lossless-flavored-markdown]]
  is actually closest to.

### [pandoc](./pandoc)
- **Repo:** https://github.com/jgm/pandoc — *the universal markup
  converter*
- **Maintainer:** John MacFarlane (`jgm`)
- **Why this is here:** The upstream authority every "Pandoc-flavored"
  claim in this study ultimately points back to. A Haskell library +
  CLI that converts between dozens of formats (CommonMark, GFM,
  AsciiDoc, DocBook, DOCX, EPUB, LaTeX, MediaWiki, reStructuredText,
  Djot, and more) through a single internal representation — parse any
  input format into it, render out to any output format, no N×M
  converter matrix. Defines the extended Markdown dialect (fenced divs,
  grid tables, definition lists, citations, inline footnotes, YAML
  metadata blocks) that Quarto and R Markdown both build on top of, and
  that [panache](./panache) explicitly treats as its own "gold
  standard" — panache vendors a copy of Pandoc's syntax spec
  (`assets/pandoc-spec.md`) as its parser's definitive reference, and
  its own contributor docs say outright: "Pandoc parser is the gold
  standard — if in doubt, see how Pandoc handles it." GPLv2+ licensed.
  The comparison this pins down: panache is a *lossless-CST, editor-
  tooling* reimplementation of Pandoc's dialect; Pandoc itself is the
  *lossy-AST, format-conversion* original the dialect is named after.

### [marksman](./marksman)
- **Repo:** https://github.com/artempyanykh/marksman — *Write Markdown
  with code assist and intelligence in the comfort of your favourite
  editor*
- **Maintainer:** Artem Pyanykh (`artempyanykh`); F#/.NET
- **Why this is here:** The **prevailing open-source Markdown LSP before
  panache**, and the natural editor-LSP comparator. Different design
  bet: where panache centers Pandoc's syntax extensions and code-block
  formatting, marksman centers **wiki-link / Zettelkasten-style
  cross-references** — completion, goto-definition, find-references,
  rename refactoring across `[[note-name]]` links and Markdown internal
  anchors. Distributed as a self-contained binary for macOS / Linux /
  Windows; available in Homebrew core. The two LSPs together define
  the axes of the editor-time design space — *syntax fidelity*
  (panache) vs. *cross-reference graph* (marksman) — and the question
  for LFM is whether we want one of these, both running side-by-side,
  or a third LSP that subsumes them.

---

## Candidates to add

Not yet pinned. When the study expands, run
`git submodule add <url> <slug>` from the study root.

### Parsers — still to add

- **micromark** — https://github.com/micromark/micromark — the
  lower-level CommonMark tokenizer that `remark` builds on; the actual
  specification-tracking parser.
- **markdown-it** — https://github.com/markdown-it/markdown-it — the
  other major JS Markdown parser; plugin-heavy, used by VuePress and
  Docusaurus-adjacent stacks.
- **pulldown-cmark** — https://github.com/pulldown-cmark/pulldown-cmark
  — Rust streaming CommonMark parser; the parser inside mdBook and
  many Rust-based tools. Token-stream rather than tree.
- **comrak** — https://github.com/kivikakk/comrak — Rust GFM-compatible
  parser with a CommonMark-faithful AST.
- **goldmark** — https://github.com/yuin/goldmark — Go's de-facto
  Markdown parser; used by Hugo. Extension model worth comparing.

### Serve / render — still to add

- **MDX** — https://github.com/mdx-js/mdx — Markdown + JSX; the third
  pole of the render-stage triangle alongside [markdoc](./markdoc) and
  Astro Content.
- **Astro Content Layer** — Astro's first-party Markdown pipeline
  (remark + rehype underneath). https://github.com/withastro/astro
- **Vale** — https://github.com/errata-ai/vale — prose linter; not a
  parser but the de-facto style/lint tool worth comparing against
  panache's linter side.

### Wikilink / directive plugins on top of remark

- **remark-wiki-link** — https://github.com/landakram/remark-wiki-link
  — wikilink plugin for remark; the `[[reference]]` syntax LFM extends.
- **remark-directive** — https://github.com/remarkjs/remark-directive
  — the `:::name` directive syntax the directive-flavored dialects
  build on.

---

## Reading order suggestion

When the study fills out:

1. **Start with panache** for the LSP + lossless-CST shape — the most
   modern take on "treat Markdown as a real language."
2. **Compare to remark + micromark** for the JS AST tradition that
   powers most of the Lossless toolchain today.
3. **Then to pulldown-cmark / comrak / markdown-rs** for the Rust
   streaming-parser tradition — relevant to how panache differs.
4. **Then to Pandoc** for the upstream dialect spec these tools track
   (or fail to track).

By the time those are read in sequence, the parse-and-serve design space
should be visible. Render-stage comparisons (MDX vs. Astro vs. Markdoc)
are a second pass.

---

## See also

- `../README.md` — the studies index, current studies list
- `../CLAUDE.md` — operational instructions for working in `studies/`
- [[lossless-flavored-markdown]] — the LFM skill, which this study grounds
- [[astro-knots]] — the family of sites that consume LFM output
