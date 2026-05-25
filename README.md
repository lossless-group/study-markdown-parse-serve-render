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

More entries to come. See *Candidates to add* below.

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
  (remark, markdown-it) that drop syntactic information at parse time.

---

## Candidates to add

These are not yet pinned as submodules. When the study expands, run
`git submodule add <url> <slug>` from the study root.

### Parsers — the canonical AST/CST players

- **remark** (unified) — https://github.com/remarkjs/remark — the dominant
  JS Markdown AST processor; powers Astro's content pipeline, MDX, and
  much of the Lossless toolchain.
- **micromark** — https://github.com/micromark/micromark — the lower-level
  CommonMark tokenizer remark builds on; the actual specification-tracking
  parser.
- **markdown-it** — https://github.com/markdown-it/markdown-it — the other
  major JS Markdown parser; plugin-heavy, used by VuePress, etc.
- **pulldown-cmark** — https://github.com/pulldown-cmark/pulldown-cmark —
  Rust streaming CommonMark parser; the parser inside mdBook and many
  Rust-based tools.
- **comrak** — https://github.com/kivikakk/comrak — Rust GFM-compatible
  parser with a CommonMark-faithful AST.
- **goldmark** — https://github.com/yuin/goldmark — Go's de-facto Markdown
  parser; used by Hugo. Extension model worth comparing.
- **Pandoc** — https://github.com/jgm/pandoc — the upstream spec for
  fenced divs, grid tables, definition lists, citations. Haskell. The
  reference for the dialect panache implements.

### Serve / render — the framework integrations

- **MDX** — https://github.com/mdx-js/mdx — Markdown + JSX; the "Markdown
  files become components" bet. Adjacent to but distinct from Astro's
  content collections.
- **Astro Content Layer** — Astro's first-party Markdown pipeline (remark
  + rehype underneath). Not a separate repo, but the canonical
  Lossless-stack render path. https://github.com/withastro/astro
- **Markdoc** — https://github.com/markdoc/markdoc — Stripe's authoring
  framework; trades JSX flexibility for tag/attribute discipline.
- **markdown-rs** — https://github.com/wooorm/markdown-rs — Rust
  CommonMark + GFM parser by the unified/remark author; the Rust answer
  to remark's design choices.

### Editor / LSP — comparators for panache

- **marksman** — https://github.com/artempyanykh/marksman — the prevailing
  open-source Markdown LSP before panache. Different bet: wikilink and
  cross-reference focused, less Pandoc-aware.
- **Vale** — https://github.com/errata-ai/vale — prose linter (not a
  parser, but the de-facto style/lint tool worth comparing against
  panache's linter side).

### Wikilink / Obsidian-flavor dialects

- **remark-wiki-link** — https://github.com/landakram/remark-wiki-link —
  wikilink plugin for remark; the [[reference]] syntax that LFM extends.
- **remark-directive** — https://github.com/remarkjs/remark-directive —
  the `:::name` directive syntax the directive-flavored dialects build
  on.

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
