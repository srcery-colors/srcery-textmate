# Aligning `srcery.tmTheme` with srcery-vim

Status: all 22 corpus languages measured; 21 aligned (every mismatch the
grammar can express is fixed, the rest are gaps in the [status table](#status)),
zsh deliberately unchanged. Remaining work is in [TODO](#todo).

## Truth hierarchy

1. **Palette** — only hexes from `@srcery-colors/srcery-palette` v2.1.0
   (`palette.json`). No exceptions.
2. **srcery-vim rendered output** — measured with the harness, never eyeballed.
3. **Existing tmTheme choices** — kept where vim has no opinion.

This repo is the canonical TextMate source. Downstream TextMate consumers
(srcery-vscode) sync from `srcery.tmTheme`; tree-sitter consumers (Zed) map the
same intent, not the selectors.

## Pinned environment

Counts in the status table are only valid against these versions. Record new
values here when bumping anything, then re-measure every language.

| Component | Version |
| --- | --- |
| srcery-palette | v2.1.0 |
| srcery-vim | `eaa0fc5` |
| Neovim | 0.12.6 (needs ≥ 0.9 for builtin `:TOhtml`) |
| bat | 0.26.1 → sublimehq/Packages `759d6ee` |
| srcery-syntax-tester (corpus + harness) | `main` @ `850f59a` |

Elixir only: ground truth also needs `elixir-editors/vim-elixir` on
`runtimepath` (srcery-vim links `elixir*` groups that only exist with it).

## Workflow

Run from a srcery-syntax-tester checkout with srcery-textmate as a sibling.

### Setup (once per session, and after every theme edit)

```bash
cp ../srcery-textmate/srcery.tmTheme "$(bat --config-dir)/themes/" && bat cache --build
export COLORTERM=truecolor
```

### Measure one language

```bash
lang=java
f=$(./scripts/list.sh | awk -v l=$lang '$1==l && $2=="valid" {print $3; exit}')
DUMP_IN=$f DUMP_OUT=/tmp/vim_$lang.html nvim --clean --headless "+luafile scripts/dump-vim-html.lua"
BAT_THEME=srcery python3 scripts/compare-bat-vim.py /tmp/vim_$lang.html "$f"
```

Output: `bat #EF2F27 -> vim #68A8E4 [Type] x20  L14:'int'; …` — bat rendered 20
tokens red; vim wants blue (`Type`). Count = sum of `xN`. Exit 1 = mismatches
exist, not a tooling failure. The vim dump only changes with vim: while
iterating, re-run just the compare step.

### Per language

1. **Measure** and save the report.
2. **Classify** each line: *theme bug* (usable scope, wrong color → fix),
   *grammar gap* (no distinguishing scope → document), *vim quirk* (match only
   if it doesn't hurt other languages).
3. **Find the scope** in the pinned grammar, never from memory:
   `curl -s https://raw.githubusercontent.com/sublimehq/Packages/759d6ee/<Lang>/<Lang>.sublime-syntax`.
   If unclear, probe: add tracer rules (`#010101`, `#010102`, …) to a copy of
   the theme, install it as a second bat theme, render a snippet with
   `bat --color=always --theme srcery-probe --style=plain <file> | cat -v`.
4. **Edit** per [rules](#theme-editing-rules). Re-measure after *each* rule.
5. **Regression-sweep** every language; counts must not rise.
6. **Commit** one language per commit:
   `fix(<lang>): align rendering with vim output`, body
   `Mismatched runs: <before> -> <after>`, one line per accepted gap, and the
   status table updated in the same commit.

## Vim group → hex

| Vim group(s) | Hex | fontStyle |
| --- | --- | --- |
| Normal | `#FCE8C3` | |
| Comment, Delimiter, LineNr | `#917E6B` | italic (Comment) |
| String | `#98BC37` | |
| Number, Constant, Boolean, Float | `#FF5C8F` | |
| Identifier, Member, StorageClass | `#68A8E4` | |
| Type | `#68A8E4` | italic |
| Function, Repeat, SpecialChar | `#FBB829` | |
| Statement, Keyword, Conditional, Exception | `#EF2F27` | |
| Include | `#F75341` | |
| PreProc, PreCondit, Structure | `#0AAEB3` | |
| Special, Tag | `#2C78BF` | |
| Operator, Label | `#C5B088` | |
| SpecialComment | `#2BE4D0` | italic |
| Define | `#FF5F00` | |
| Decorator/Annotation, Todo | `#FF8700` | |
| Typedef | `#E02C6D` | |
| Error | `#FCE8C3` on `#EF2F27` | bold |

## Theme editing rules

- **Specificity is leaf-length first**; file order only breaks ties. If a rule
  doesn't take, lengthen its leaf.
- **Selectors match per-segment prefixes**: `keyword.operator.js` does not
  match `keyword.operator.arithmetic.js`.
- **Always keep the language suffix**, on every listed variant. An unsuffixed
  prefix leaks into other languages (erlang's `constant.other.placeholder`
  recolored Ruby).
- **Never change a generic rule to fix one language.**
- **When a grammar co-assigns scopes, select the most specific one**
  (go: `support.function.builtin.go`, not shared `variable.function.go`).
- **Prefer leaf scopes over direct `meta.*`.** Approved direct-meta selectors
  (verified present, and the only ones, on `master`):
  `meta.object-literal.key`, `meta.object-literal.key.ts`,
  `meta.brace.square.ts`, `meta.property.lua`, `meta.function-call.name.fish`,
  `meta.preprocessor.haskell`, `meta.table.header.markdown-gfm`,
  `meta.link.reference.description.markdown`. Adding one needs a documented
  grammar limitation.
- **fontStyle** is `bold`/`italic`/`underline` or `""` to cancel. **Rule names**
  state intent and vim group: `Java: doc comment (vim Comment)`.

## Established decisions

1. **Operator split** — symbolic `keyword.operator` white `#C5B088`; word-like
   (`.word`, `.expression`, bare `keyword.operator.js`/`.ts`) red `#EF2F27`.
2. **`keyword.declaration`, `storage.type` → red** (Statement); per-language
   overrides for primitives (C, Java) instead of flipping the base rule.
3. **Call sites are plain** — no generic `variable.function` rule.
4. **`variable.language` stays foreground** generically; per-language
   overrides where measured.
5. **Gutter, brackets, active guide → white `#C5B088`** for readability
   (8.93:1 on `#121110`), not vim's `LineNr` `#917E6B` (4.85:1).
6. **Background `#121110`.**
7. **Doc comments** `#2BE4D0` italic generically; per-language overrides
   where vim differs (python green, java gray).
8. **Bat beats vim where vim is cruder by design**: embedded JS in HTML and
   fenced code in markdown stay highlighted by bat's injection.
9. **Elixir** is measured with vim-elixir loaded (only exception to the
   vanilla recipe).

## Known false positives

Check these before chasing a mismatch:

- **Multi-line regions**: `:TOhtml` spans only the first/last line of a block
  comment or long string. Confirm with `bat --color=always … | cat -v`.
- **Run majority**: the compare script reports a run's majority color; a
  partially fixed token can still show as mismatched.
- **Treesitter filetypes** (help, lua, markdown, query): `:TOhtml` shows
  treesitter captures on top of legacy syntax. Diagnose with
  `nvim_get_hl(0, {name='@capture'})` and `$VIMRUNTIME/queries/<lang>/`, not
  `synID`. Re-check this list when bumping Neovim:
  `grep -l treesitter.start $VIMRUNTIME/ftplugin/*.lua`.
- **Shared scopes**: before calling something a gap, check for an extra
  co-assigned scope or ancestor to select on (CSS `.class`/`#id`, PowerShell
  `meta.attribute`/`meta.function`). Before widening a rule, check the scope
  isn't shared with a commoner construct (rust closure pipes vs fn parens).
- **Dead rules**: an existing selector may target a scope the pinned grammar
  never emits. Verify before building on it.

## Status

Baseline = mismatched runs at the pinned environment. Gap details live in each
language's commit message.

| Language | Baseline | Main remaining gaps |
| --- | --- | --- |
| bash | 30 | array keys, RHS `$((`, cmd flags/args, external commands |
| python | 6 | `match`/`case`, `self` in defs, `_` wildcard |
| java | 37 | rainbow parens (~28), text blocks, `record`, `.class` |
| typescript | 50 | decl vs reference (~16), vim builtin-method keyword list (~12), shared `=` |
| rust | 77 | unscoped path segments/derives/`Type::fn` (~34), `&`/`let`/`->` shared scopes |
| go | 8 | `for`/`case`/`default` share `keyword.control.go`; return/closure param types |
| c | 8 | `storage.type.c` primitives vs typedef/struct; macro bodies; `for` |
| css | 21 | grammar predates `gap`, range media, `font-display`; `calc()` internals |
| html | 39 | embedded JS (decision 8, ~35); per-tag content groups |
| yaml | 1 | `3.0.0` shares float scope |
| lua | 17 | ALL_CAPS `@constant` regex (8), loop `in`/`end`, 7 multi-line artifacts |
| ruby | 20 | kwargs vs positional params; shared `end` |
| zsh | 62 | **unchanged on purpose**: shares bash grammar, vim zsh.vim disagrees with sh.vim |
| fish | 27 | most builtins unscoped (~22) |
| powershell | 27 | hashtable keys vs variables (18), unscoped `-Param` names (6) |
| haskell | 10 | `import`/`let`/`in` share `keyword.other`; `*` unscoped |
| markdown | 4 | task-list brackets; fenced code (decision 8); H3–H6 untested |
| clojure | 28 | special forms share call-position scope (23) |
| elixir | 12 | typespec `::` unscoped (7), `@moduledoc` single scope, `_var` regex |
| erlang | 0 | — |
| scala | 5 | interpolated quotes, `_` pattern, Scala 3 lambda arrow, `%-8s` |
| json | 1 | `null` shares `constant.language.json` with booleans |

## Open decisions

Each needs a maintainer answer, then a line in [decisions](#established-decisions).

- **zsh vs bash** priority on shared `source.shell` scopes. Default until
  decided: bash wins.
- **Gutter color across consumers**: srcery-vscode uses `#917E6B`; this repo
  uses `#C5B088` after the gutter fix. Pick one.
- **srcery-syntax-tester visibility**: private blocks contributors and CI in
  this repo.

## TODO

Each task ends when its **Done** condition holds.

1. **Re-measure all 22 languages at the pinned environment.** java,
   typescript and rust predate both harness fixes.
   Done: status table counts confirmed or updated in one commit, with any
   changed gap re-audited.
2. **Pin srcery-vim in the harness** (`dump-vim-html.lua` honours a ref
   instead of cloning HEAD).
   Done: dump output identical across two fresh runs with the env var set.
3. **Add baseline mode to `compare-bat-vim.py`** (`--max N` or a baselines
   file; exit 0 unless count exceeds baseline).
   Done: script exits 0 for all 22 languages on `master`.
4. **Add `scripts/lint-theme.py`** here: plist parses, every hex in
   `palette.json` v2.1.0, fontStyle values valid, direct-meta selectors in
   the approved list.
   Done: passes on `master`, fails on a planted off-palette hex.
5. **CI workflow** running tasks 3 and 4 on PRs. Depends on the visibility
   decision.
   Done: required status check green on `master`.
6. **Add a JavaScript sample** to the corpus and align it.
   Done: javascript row in status table.
7. **Add a diff/patch sample** and align `markup.inserted/deleted/changed`
   with vim `diffAdded`/`diffRemoved`/`diffChanged`.
   Done: diff row in status table.
8. **Add `gutterForegroundHighlight` `#FBB829`** (vim `CursorLineNr`).
   Done: key present, verified in Sublime Text.
9. **Remove the duplicate `support.class` selector.**
   Done: no selector appears in two rules.
10. **VS Code scope spike**: tokenize the corpus with `vscode-textmate` +
   VS Code's bundled grammars under this theme; list rules that never match.
   Done: report attached to an issue in srcery-vscode.
11. **srcery-syntax-tester housekeeping**: README language list (22), MANIFEST
   count, delete merged branch `add-vim-truth-comparison`.
   Done: counts consistent; branch gone.
