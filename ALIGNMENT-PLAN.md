# Plan: align `srcery.tmTheme` with srcery-vim, language by language

Status: **bash ✓, python ✓** (done on this branch, see commit `5618bc7`).
This document is a self-contained runbook for aligning the remaining corpus
languages. It assumes no prior session context — everything needed is here or
in the referenced repos. Work through one language at a time, top to bottom of
the [Language queue](#language-queue).

## Goal and truth hierarchy

`srcery.tmTheme` (this repo) is a TextMate/Sublime port of the Srcery color
scheme, used by bat, Sublime Text, and Textastic. The reference implementation
is **srcery-vim**; when the two disagree, vim wins, in this order:

1. **Palette** — only hexes from `@srcery-colors/srcery-palette` v2.0.0 may
   appear in the theme (plus the intentional `#D0BFA1` gutter deviation, see
   [Established decisions](#established-decisions-do-not-re-litigate)).
2. **srcery-vim rendered output** — what real vim shows for a token, measured
   (never eyeballed) with the harness below.
3. **Existing tmTheme choices** — kept when vim expresses no opinion.

"Aligned" does **not** mean zero mismatches. TextMate grammars and vim syntax
files tokenize differently; some tokens carry no usable scope at all. Aligned
means: every mismatch that the grammar *can* express is fixed, and the rest
are documented as gaps.

## Prerequisites

- This repo, on branch `sync-palette-with-srcery-vim` (PR #3).
- `srcery-colors/srcery-syntax-tester` cloned as a sibling
  (`~/Developer/srcery-syntax-tester` in the original run), on branch
  `add-vim-truth-comparison` (PR #1) — that branch has the two harness
  scripts. If PR #1 has been merged, `main` is fine. **Make sure you have
  commit `26c9665` or later** (two harness bugs fixed since the original
  scripts were written):
  - `1ee2502`: `compare-bat-vim.py`'s CSS `color:` regex also matched
    inside `background-color:`, corrupting results whenever a `:TOhtml`
    class set both on one rule — only ever observed on typescript's
    `.Normal` rule.
  - `26c9665`: `dump-vim-html.lua` didn't force `tabstop=1`, so `:TOhtml`
    rendered tabs at display width (8 columns) while bat/syntect kept the
    literal `\t` as one character — every tab-indented sample (go's whole
    file; a few of bash's heredoc lines) was badly column-misaligned past
    the first tab. Space-indented samples were unaffected.
  Re-verify bash/python/java counts against the queue table after pulling
  to be sure your checkout has both fixes.
- **Known, not-yet-fixed harness limitation** (found during the lua pass,
  no commit yet): `:TOhtml` only wraps the *first and last* physical line
  of a multi-line highlighted region (block comments, long strings) in a
  `<span>` — continuation lines in between carry no span at all. Both the
  line-by-line HTML parser in `compare-bat-vim.py` and any manual
  per-line dump read that as "vim wants this plain." It doesn't; bat is
  usually already rendering those lines correctly and the apparent
  mismatch is pure measurement noise. Before spending time on a
  multi-line comment/string mismatch, verify with raw `bat --color=always
  ... | cat -v` on the same lines — if bat is already internally
  consistent (every continuation line the same color), it's this
  artifact, not a real gap. Not worth fixing in the harness yet (narrow,
  and every language measured so far already renders correctly); revisit
  if it starts hiding real issues.
- Neovim ≥ 0.9 (`:TOhtml` builtin; the dump script `packadd`s it).
- `elixir-editors/vim-elixir` cloned to a scratch dir, **only if
  re-measuring the elixir row** — see the elixir methodology-exception
  note under the queue table. No other row needs it.
- `bat` (the original run used 0.26.1 — the pinned grammar revision below is
  tied to the bat version; re-derive it if bat differs).
- `python3`, `curl`, `plutil` (macOS) or python `plistlib` for linting.

One-time setup per session:

```bash
# Install the theme under test into bat and build the cache.
cp srcery.tmTheme "$(bat --config-dir)/themes/srcery.tmTheme"
bat cache --build
export COLORTERM=truecolor
```

**Re-run `cp` + `bat cache --build` after every theme edit.** Forgetting this
is the #1 source of confusing results.

Then, before touching anything, re-measure the already-done languages and
save the output — these are your session baselines for regression checks
(counts should match the queue table; if they don't, your environment
differs — stop and investigate before editing):

```bash
for lang in bash python; do
  f=$(./scripts/list.sh | awk -v l=$lang '$1==l && $2=="valid" {print $3; exit}')
  DUMP_IN=$f DUMP_OUT=/tmp/vim_$lang.html \
    nvim --clean --headless "+luafile scripts/dump-vim-html.lua"
  BAT_THEME=srcery python3 scripts/compare-bat-vim.py /tmp/vim_$lang.html $f \
    | tee /tmp/baseline_$lang.txt
done
```

`scripts/list.sh` prints every sample as `<lang> <validity> <path>` — use it
for exact file paths rather than trusting the queue table's guesses.

## The harness

Both scripts live in srcery-syntax-tester `scripts/`.

```bash
cd ~/Developer/srcery-syntax-tester

# 1. Ground truth: render the sample through real srcery-vim (vanilla, via
#    nvim --clean; auto-detects or auto-clones srcery-vim) and dump HTML.
DUMP_IN=languages/<lang>/valid/<sample> DUMP_OUT=/tmp/vim_<lang>.html \
  nvim --clean --headless "+luafile scripts/dump-vim-html.lua"

# 2. Diff bat's render of the same file against that HTML.
BAT_THEME=srcery python3 scripts/compare-bat-vim.py \
  /tmp/vim_<lang>.html languages/<lang>/valid/<sample>
```

The compare script exits **1 when mismatches exist** (0 when in sync) — an
exit-1 during iteration is expected, not a failure of the tooling.

Output format — one line per distinct `(bat color, vim color, vim group)`
triple, sorted by frequency, with up to 5 example tokens:

```
bat #EF2F27 -> vim #68A8E4 [Type] x20  L14:'int'; L16:'double'; ...
```

Read it as: *bat rendered these 20 tokens `#EF2F27`; vim renders them
`#68A8E4` via highlight group `Type`.* The group name is the key signal — it
tells you srcery-vim's intent without opening srcery.vim.

The vim HTML dump does not change when you edit the theme, so dump once per
language and re-run only step 2 while iterating.

## Vim group → hex quick reference

All from srcery-vim / palette v2.0.0. `fontStyle` column is what the tmTheme
should use when creating a rule for that role.

| Vim group(s) | Hex | fontStyle |
| --- | --- | --- |
| Normal | `#FCE8C3` | |
| Comment, Delimiter, LineNr | `#917E6B` | italic (Comment only) |
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
| Error | `#FCE8C3` on bg `#EF2F27` | bold |

Language-specific vim groups (e.g. `javaParen1`, `shVarAssign`) show up in
the diff with their raw names; their color in the report is still the final
resolved hex, which is all you need.

## Per-language workflow

For each language in the queue:

### 1. Measure

Run the harness (above). Save the initial report — it goes in the commit
message as the "before" count. Count = sum of the `xN` values.

### 2. Classify every mismatch line

Three buckets:

- **Theme bug** — the grammar emits a usable scope but the theme maps it to
  the wrong color (or doesn't map it). Fix it.
- **Grammar gap** — syntect/bat's grammar emits no distinguishing scope for
  the token (e.g. bash array-literal keys, python `match`/`case`). Document
  it; do not chase it with hacks.
- **Vim quirk** — vim highlights something TextMate conventions say to leave
  alone, or inconsistently (e.g. vim java colors *every* paren via
  `javaParen1` rainbow groups). Judgment call: match vim only if it doesn't
  damage other languages; otherwise document as an accepted deviation.

### 3. Resolve the scope for each theme bug

Never guess scopes from memory — the grammar bat actually bundles is **not**
sublimehq/Packages master. Pin it:

```bash
# Find the Packages submodule revision for the installed bat version:
curl -s "https://api.github.com/repos/sharkdp/bat/contents/assets/syntaxes?ref=v$(bat --version | cut -d' ' -f2)" \
  | python3 -c "import json,sys; print([f['sha'] for f in json.load(sys.stdin) if f['name']=='01_Packages'][0])"
# (v0.26.1 -> 759d6eed9b4beed87e602a23303a121c3a6c2fb3)

# Fetch the language's grammar at that revision and grep for scopes:
curl -s -o /tmp/Java.sublime-syntax \
  "https://raw.githubusercontent.com/sublimehq/Packages/<REV>/Java/Java.sublime-syntax"
grep -nE "scope:|meta_scope|meta_content_scope" /tmp/Java.sublime-syntax | grep -i <construct>
```

Some grammars `include: scope:another.syntax` — fetch that file too (e.g.
bash builtins live in `commands-builtin-shell-bash.sublime-syntax`). Some
languages ship in bat's `assets/syntaxes/02_Extra` instead of Packages.

When the grammar is ambiguous or the fix doesn't take, **probe empirically**:
append tracer rules to a *copy* of the theme, each candidate scope mapped to
a unique fake color (`#010101`, `#010102`, …), install it as a second bat
theme, render a minimal snippet, and read which tracer color each token got:

```bash
python3 - <<'PY'
import plistlib
d = plistlib.load(open('srcery.tmTheme','rb'))
for i, scope in enumerate(['candidate.scope.one', 'candidate.scope.two']):
    d['settings'].append({'name': f'PROBE {scope}', 'scope': scope,
                          'settings': {'foreground': f'#0101{i:02X}'}})
plistlib.dump(d, open('/tmp/srcery-probe.tmTheme','wb'))
PY
cp /tmp/srcery-probe.tmTheme "$(bat --config-dir)/themes/" && bat cache --build
bat --color=always --theme srcery-probe --style=plain <snippet> | cat -v
# Delete the probe theme + rebuild cache when done.
```

### 4. Edit the theme

Rules of the road (violating these caused most of the bugs during the
bash/python passes):

- **Specificity is leaf-length first.** The rule whose selector matches the
  token's *deepest* scope element with the *longest* segment prefix wins;
  rule order in the file is only a tie-break. A plain 5-segment selector
  beats a descendant selector with a 3-segment leaf. If your new rule
  doesn't take effect, lengthen its leaf (e.g.
  `meta.group.expansion.parameter keyword.operator.assignment.shell`, not
  `... keyword.operator.assignment`).
- **Scope selectors are per-segment prefixes.** `keyword.operator.js`
  matches `keyword.operator.js` but *not* `keyword.operator.arithmetic.js`
  (segment 3 differs). This is how you can target JS's bare-scoped
  `typeof`/`instanceof` without touching symbolic operators.
- **Prefer language-suffixed rules** (`....python`, `....shell`) or
  `source.<lang>` descendant selectors for language-specific vim behavior.
  Never change a generic rule to fix one language — check what else uses it.
- **No bare `meta.*` in scope selectors** (styling meta scopes is against
  TextMate conventions). Exceptions already granted: `meta.object-literal.key`.
  Using `meta.X` as the *ancestor* in a descendant selector is fine.
- **Only palette hexes.** If vim truth needs a color, it's in the quick
  reference above. Never invent one.
- **`fontStyle` values**: exactly `bold`, `italic`, `underline`, space-
  separated combos, or empty string (to cancel an inherited style). No
  leading/trailing spaces. No `strikethrough`.
- Name new rules descriptively and note the vim group:
  `<string>Java: doc comment (vim Comment)</string>`.

### 5. Re-measure and iterate

`cp` + `bat cache --build` + re-run step 2. Repeat 2–5 until every remaining
line is a classified gap/quirk. Expect 2–4 iterations per language.

### 6. Regression-check ALL previously aligned languages

Theme rules are global. After the language passes, re-run the compare script
for **every language already done** (bash, python, plus everything above the
current one in the queue) and confirm the counts match the accepted-gap
baselines recorded in the queue table. Also run the JS operator smoke test:

```bash
printf 'if (typeof x instanceof Foo) {\n  let y = a + b;\n}\n' > /tmp/probe.js
bat --color=always --theme srcery --style=plain /tmp/probe.js | cat -v
# typeof/instanceof red #EF2F27; let/if red; = and + white #C5B088.
```

And lint: `plutil -lint srcery.tmTheme` (or
`python3 -c "import plistlib; plistlib.load(open('srcery.tmTheme','rb'))"`).

### 7. Commit

One commit per language on `sync-palette-with-srcery-vim`:

```
Align <lang> rendering with vim output

Mismatched runs: <before> -> <after>. Remaining are grammar gaps:
<one line per accepted gap>.
<bullet list of the actual color changes, with vim group names>
```

Update the queue table in this file in the same commit. After a batch of
languages, post a summary comment on PR #3 (`gh pr comment 3 --body ...`)
with the per-language before/after table.

## Established decisions (do not re-litigate)

These came out of maintainer review (@roosta) and the bash/python passes.
Re-opening them will regress other languages:

1. **Operator split** — symbolic operators (`keyword.operator` base) are
   white `#C5B088`; word-like operators (`keyword.operator.word`,
   `.expression`, bare `keyword.operator.js`/`.ts`) are red `#EF2F27`.
   History: roosta asked for red keywords (JS `typeof`); all-red broke
   `=`/`;`/`+` vs vim. The split satisfies both.
2. **`keyword.declaration, storage.type` → red** (vim Statement): `def`,
   `class`, `function`, `var`, `declare`… `storage.type.c` is overridden to
   blue-italic (vim Type for C primitives). Expect similar per-language
   overrides (java's `int`/`double` — see appendix) rather than flipping the
   base rule.
3. **Call sites are plain** — vim doesn't color function calls; the
   `variable.function` rule was removed. Don't re-add it.
4. **`variable.language` (this/self/super) stays foreground** as a generic
   rule; language-specific overrides (e.g. `variable.language.python` blue)
   are fine where measured.
5. **`#D0BFA1` on gutterForeground/bracketsForeground/activeGuide** is an
   intentional readability deviation (commit 39f4efa). Leave it.
6. **Background `#121110`** (current srcery black), not classic `#1C1B19`.
7. **Doc comments** are generically `#2BE4D0` italic (vim SpecialComment,
   matches rust `///`). Python overrides to green (vim String). Other
   languages: measure and override per-language if vim disagrees (java: vim
   says Comment-gray body with white title — see appendix).

## Known gap classes (recognize and move on)

- Tokens the grammar leaves scopeless: bash array-literal keys `[red]=`,
  `$((…))` on assignment RHS, flags after generic commands, external command
  names, bare numeric arguments.
- Grammar predates the syntax: python `match`/`case` (grammar is pre-3.10).
- Context vim can see but the grammar can't: python `self` inside `def`
  signatures (scoped as a plain parameter).
- Vim rainbow/pair quirks: `javaParen1`, vim's `$` vs `${` distinctions.
- Run-majority artifacts: the compare script reports a run's majority color,
  so a 2-char token with 1 fixed + 1 unfixable char may still show as
  mismatched (`$1` in bash: `$` cyan, `1` blue). Confirm by reading the raw
  render (`bat --color=always ... | cat -v`) before chasing.
- No semantic reference tracking: static TextMate grammars scope a
  declaration (e.g. a function parameter or const binding) but usually
  cannot scope every later *usage* of that same identifier differently from
  an unrelated identifier. If vim colors declarations and usages
  differently (found repeatedly in TypeScript: parameters/locals are
  `PreProc` cyan at every occurrence in vim, but the grammar only scopes the
  declaration), only the declaration is fixable.
- Keyword-listed builtin methods: some vim syntax files hardcode a list of
  "well-known" API method names (e.g. TS's `.map`/`.test`/`.set`/`.log`) and
  color them like keywords. The grammar has no way to distinguish these
  from any other method call — same scope either way. Not fixable.
- Path/module segments with no scope at all: some grammars simply don't
  assign a scope to certain positions (e.g. rust's `std`/`collections` in
  `use std::collections::X`, or a type name used as a path prefix like
  `Tone::Dark`). Confirm by reading the grammar context directly — if the
  context has no bare-identifier match at all (only literal keywords or
  punctuation), there is nothing to style. Same root cause explains why
  `Type::associated_fn()` call sites can't be colored like other function
  calls in some grammars.
- **Before generalizing a scope-color change, verify it doesn't hit a more
  common construct with a different colored elsewhere.** A rust fix mapping
  `punctuation.section.parameters.begin/end.rust` (closure `|params|`
  pipes) to Operator-white also recolored ordinary function-declaration
  parens, which use the identical scope — net negative (broke a common
  case to fix a rarer one). Re-measure immediately after each individual
  rule add, not just at the end of a batch, so a regression like this is
  caught and reverted before it's buried under later changes.
- **When a grammar co-assigns two scopes to one token (`name: scope.a
  scope.b`), prefer the more specific/unique one as your selector**, not
  whichever comes first when you eyeball the source. Go's builtin
  functions (`len`, `make`, `append`) get `variable.function.go
  support.function.builtin.go` together, but plain user-defined calls get
  bare `variable.function.go` alone. Targeting the shared half
  (`variable.function.go`) recolors both; targeting the builtin-only half
  (`support.function.builtin.go`) doesn't. Same root cause as the rust
  lesson above — re-measure right after adding the rule to catch it.
- **When you suspect a scope is shared between constructs vim colors
  differently, confirm it by directly probing vim rather than reasoning
  from the sample file alone.** For C, `if`/`for`/`switch`/`case`/
  `default` all reduce to one `keyword.control.c` in the grammar; rather
  than guessing whether vim also treats them identically, a two-second
  throwaway snippet (`if/else/switch/case/default/while/for` in one file,
  dumped and read back) confirmed vim gives each its own color
  (Conditional/Repeat/Label) — settling definitively that the scope really
  is unfixable, not just untested in that particular sample.
- **A scope shared three-plus ways isn't always a dead end — check
  whether each instance has an *additional*, more specific co-assigned
  scope you can layer a descendant selector on top of.** CSS's class-dot,
  id-hash, and attribute-selector bracket all reduce to the same bare
  `punctuation.definition.entity.css` capture, wanting three different
  colors. But the dot is *also* nested under `entity.other.attribute-
  name.class.css` and the hash under `...id.css` — selectors combining
  both (`entity.other.attribute-name.class.css punctuation.definition.
  entity.css`) resolved the dot and hash independently, leaving only the
  bracket (which has no such extra ancestor) on the bare fallback. Don't
  assume a shared scope is automatically a gap; check what else is
  co-assigned to the specific token first.
- **Rules added in earlier passes can themselves be wrong, not just
  incomplete — verify they still match the *actual* bat grammar before
  building on them.** Three CSS rules from the original pre-alignment
  P1/P2 pass targeted scope names (`variable.other.custom-property.css`,
  `entity.other.attribute-name.pseudo-class.css`) that this bat grammar
  version never emits at all — they'd been silently dead the whole time.
  When a measured mismatch doesn't match your mental model of "this
  should already be fixed," check whether an existing rule's selector is
  simply wrong, not just that a new override is needed.
- **Not every vim/bat mismatch means bat is wrong — sometimes vim's own
  highlighting is genuinely cruder, and matching it would be a
  regression.** In `html/valid/palette.html`, vim's embedded-`<script>`
  handling falls back to one uniform Special-blue for almost everything
  beyond a couple of literal keywords, while bat's real grammar injection
  correctly re-parses the block as JavaScript (arrow functions,
  destructuring, template interpolation, all distinctly colored). The
  corpus README explicitly states the goal for this file is proper
  CSS/JS coloring, not undifferentiated text — so degrading bat to match
  vim here would work against the corpus's own stated intent. When you
  hit this, check the corpus's README/goals for that file before
  "fixing" toward vim; sometimes the target is deliberately bat's
  behavior, not vim's.
- **Some filetypes' ground truth is treesitter, not legacy `:syntax` —
  check `$VIMRUNTIME/ftplugin/<lang>.lua` before trusting `synID`.** For
  markdown, `synID`/`synstack` on a bare `:edit` + `filetype detect` +
  `syntax sync fromstart` session returned *nothing* (empty stack, even
  on an obviously-colored heading) — because Neovim's bundled
  `ftplugin/markdown.lua` calls `vim.treesitter.start()` unconditionally,
  and `:TOhtml` renders from the *composited* result (legacy syntax
  region **and** treesitter capture stacked, treesitter on top and
  visually winning). `synID(l, c, 1)` alone only sees the legacy layer,
  so probing it directly gave answers that flatly contradicted the real
  `:TOhtml` dump (e.g. `markdownListMarker` legacy scope said Keyword/red
  for a list bullet, but the actual rendered span was Special/blue from
  the treesitter `@markup.list` capture nested on top). Once
  `vim.api.nvim_get_hl(0, {name='@capture.name'})` was queried directly
  instead, everything lined up. **This affects at least `help`, `lua`,
  `markdown`, and `query` filetypes** (grep
  `$VIMRUNTIME/ftplugin/*.lua` for `treesitter.start` to check others) —
  the already-"done" **lua** pass (222 → 21) predated this discovery and
  was diagnosed using legacy-`:syntax`-only reasoning throughout.
  **Revisited 2026-07-07: the concern was justified.** Re-auditing all
  21 gaps against `$VIMRUNTIME/queries/lua/highlights.scm` and the
  actual grammar source found 2 of the 5 gap classes (4 tokens) had
  been misdiagnosed as "no distinguishing scope" when perfectly usable
  scopes existed (`constant.language.lua` for vararg `...`,
  `support.constant.builtin.lua` for stdlib module names) — fixed,
  21 → 17. The other 3 classes were confirmed unfixable, but for
  treesitter reasons, not the legacy reasons originally recorded (see
  the lua queue row). Lesson: a "no scope exists" claim is only as good
  as the grammar-source read that produced it — when a pass's
  methodology is later found flawed, re-audit its *negative* findings
  (accepted gaps), not just its positive ones; fixes get re-verified by
  every regression sweep, but a wrong "unfixable" verdict silently
  persists forever.
- Neovim's bundled markdown highlight query
  (`$VIMRUNTIME/queries/markdown{,_inline}/highlights.scm`, from
  MDeiml/tree-sitter-markdown) captures list markers, table
  pipes/borders, inline-code spans, and block quotes all under a small
  set of generic captures (`@markup.list`, `@punctuation.special`,
  `@markup.raw`, `@markup.quote`) that srcery-vim links uniformly to
  `Special` (blue) — a single TextMate-side color fixes all of them at
  once, which is why this pass's fix-to-gap ratio was unusually high
  (12 → 3).
- `@markup.quote` wraps the **entire** block-quote node (marker *and*
  prose text) as one blue span, not just the `>` marker — confirmed via
  the raw HTML dump, not assumed. Matching this literally (`markup.quote.
  markdown` → blue, covering the whole quoted paragraph) is correct per
  the ground-truth hierarchy even though it looks unusual; this isn't a
  "vim is cruder" case, it's vim's actual intended rendering.
- Reconfirmed the multi-line-region `:TOhtml` limitation (see harness
  notes above) on a new construct: a 3-line lazy-continuation blockquote
  showed line 1 and line 2 rendering correctly, but line 3 (a
  continuation line) reported a false "wants plain" mismatch purely
  because `:TOhtml` never re-opens the unclosed multi-line span on
  continuation lines. Verified via direct `bat --color=always | cat -v`
  that all three lines already render identically (correctly) before
  concluding it was a harness artifact, not a real gap.
- The Sublime Markdown grammar only distinguishes `markup.heading.1.
  markdown` and `markup.heading.2.markdown`; H3–H6 all collapse into one
  generic `markup.heading.markdown` with no per-level scope. srcery-vim
  gives H1–H6 six genuinely distinct colors (SrceryH1..H6). Only H1
  (coincidentally already matching the flat legacy rule) and H2 (fixed
  this pass) are distinguishable in bat's theme — H3/H4/H5/H6 are an
  unfixable grammar-vintage gap, untested here since the sample has no
  H3+ headings.

## Language queue

Order: by likely usage, markup last. Update `after` counts + gap notes as
languages complete. `before` counts unknown until measured.

| # | Language | Sample | Status | Runs before → after | Accepted gaps |
| --- | --- | --- | --- | --- | --- |
| – | bash | `languages/bash/valid/palette.sh` | **done** | ~170 → 30 (re-measured 2026-07-07; originally recorded as 31 before the two harness fixes landed) | array keys, RHS `$((`, cmd flags, `cat`, bare args |
| – | python | `languages/python/valid/palette.py` | **done** | ~90 → 6 | `match`/`case`, `self` in defs, `_` wildcard |
| 1 | java | `languages/java/valid/Palette.java` | **done** | 143 → 37 | rainbow-paren/bracket nesting depth (~28), text blocks + `.formatted` chain, `record` keyword, switch-expr arrow-form `default`, `.class` literal, javadoc first-sentence title styling |
| 2 | typescript | `languages/typescript/valid/palette.ts` | **done** | 247 → 50 | reference-vs-declaration has no distinguishing scope (~16), vim keyword-lists common builtin method names as pseudo-keywords (~12), scope reused for two different vim intents e.g. type-alias `=` vs value `=` (~6), destructuring-bracket/regex-internals edge cases (~6), brace/bracket majority trade-off (2) |
| 3 | rust | `languages/rust/valid/palette.rs` | **done** | 158 → 77 | path segments before `::` have no scope at all (~18), `&`/`&mut` share a scope with correctly-white operators (~11), `let` shares a scope with const/static/primitives (~9, kept the primitives fix as majority), derive-macro trait names unscoped (~8), `Type::method()` call-site names unscoped (~8), `->`/closure `\|params\|` share scopes with plain separators/parens (~14, one fix tried and reverted for net regression), `self`-as-parameter indistinguishable from other params (~3), anonymous lifetime `'_`, for-loop `in`, array-length integer literals |
| 4 | go | `languages/go/valid/palette.go` | **done** | 37 → 8 (see harness-bugfix prerequisite above — the raw first measurement was ~130, mostly tab-misalignment noise) | `for`/`case`/`default` share `keyword.control.go` with correctly-red `if`/`switch` (5), a type name used as a *return type* or *closure param type* specifically stays uncolored while every other type-reference position is now fixed (3) |
| 5 | c | `languages/c/valid/palette.c` | **done** | 31 → 8 | `storage.type.c` shared between primitives (blue, majority ~5:1) and typedef/struct/enum (want cyan) (4); function-like macro *body* is one flat Define-orange span in vim but semantically parsed C tokens in the grammar (3); `for` shares `keyword.control.c` with correctly-colored if/switch/case/default, confirmed via direct probe (1) |
| 6 | css | `languages/css/valid/palette.css` | **done** | 163 → 21 | grammar predates the `gap` shorthand, CSS4 media range syntax (`width >= 40rem`), `font-display: swap`, and distinguishing generic vs. arbitrary font-family names; arithmetic inside `calc()`/`rgb()` has no distinguishing scope; `transform` as a transition-property *value* isn't distinguished from any other keyword; a few narrow single-occurrence paren/comma cases |
| 7 | html | `languages/html/valid/palette.html` | **done** | 126 → 39 | vim's embedded-JS highlighting is a coarse Special-blue fallback, not real dispatch to javascript.vim — bat's genuine grammar injection is *more* correct and intentionally left unmatched, per the corpus README's own stated goal (~35); vim's Link/Title/SrceryH1/H2 groups do real per-tag content matching with no TextMate scope equivalent (4); CSS font-name gap carried over from the CSS pass (1) |
| 8 | yaml | `languages/yaml/valid/palette.yaml` | **done** | 115 → 1 | `3.0.0` shares its numeric scope with legitimate floats like `2.2` (confirmed via tracer probe) — fixing it would recolor every real float |
| 9 | lua | `languages/lua/valid/palette.lua` | **done** (revisited 2026-07-07 under the treesitter lens) | 222 → 21 → 17 | The original pass's "no distinguishing scope" claims for vararg `...` and stdlib-module refs were **wrong** — legacy-`:syntax`-era diagnosis, never re-checked against the grammar source. `...` gets its own bare `constant.language.lua` (booleans/nil use longer distinct scopes) and `string`/`math`/etc. get `support.constant.builtin.lua`; both now fixed to Special blue (4 tokens recovered). Remaining 17: ALL_CAPS locals are a treesitter *naming-convention regex capture* (`@constant` on `^[A-Z][A-Z_0-9]*$`), while the grammar scopes them as plain `variable.other.lua` — genuinely inexpressible (8); for-in's `in` shares `keyword.control.loop.lua` with for/while/until which correctly stay yellow, though treesitter wants `in` as plain red `@keyword` (1); every `end` gets one uniform `keyword.control.end.lua` regardless of which block it closes, while treesitter's per-construct captures want for-loop `end` yellow — majority (if/function `end`, red) kept (1); 7 are the known multi-line `:TOhtml` span artifact on the block comment and long string — bat already renders those lines correctly |
| 10 | ruby | `languages/ruby/valid/palette.rb` | **done** | 114 → 20 | method-signature keyword params share a scope with ordinary positional params in this file (confirmed, not inferred); do-block/case `end` share a scope with the now-correct def/class/module `end`; shebang, magic-comment split, heredoc marker, `.freeze`, `Hash.new`, namespaced `Module::CONST` are narrow single-occurrence cases |
| 11 | zsh | `languages/zsh/valid/palette.zsh` | **done (no changes)** | ~50 → ~50, no theme edits made | see note below the table — most mismatches are a bash-vs-zsh vim dialect conflict, not fixable without regressing the completed bash work |
| 12 | fish | `languages/fish/valid/palette.fish` | **done** | 59 → 27 | most fish builtins (set/test/echo/end/function/for/switch/case) carry no scope at all in this grammar, confirmed via tracer probing rather than assumed (~22); the one keyword scope that does exist is shared 3-way with call-site and declaration-site function names, kept for keywords since they outnumber the others (~5) |
| 13 | powershell | `languages/powershell/valid/palette.ps1` | **done** | 78 → 32 | hashtable keys share a scope with `$variable` refs (majority kept); `function` shares a scope with type annotations (majority kept); ForEach-Object/Where-Object are hardcoded into the same match as ordinary cmdlets; `foreach` shares the control-keyword scope with if/switch/else/default; `$_` (`variable.language.powershell`) and `[int[]]` (`storage.type.powershell`) resolve to real scopes but the grammar co-assigns them to groups shared with unrelated tokens, so they inherit the wrong color; `.SYNOPSIS`/`.DESCRIPTION` help tags — resolved: the grammar splits the token into a base `comment.documentation.embedded.powershell` scope plus nested captures (`constant.string.documentation.powershell` for the dot, `keyword.operator.documentation.powershell` for the keyword); vim's `ps1CommentDoc` colors the whole `.SYNOPSIS` token as one unit (`Tag`), so all three scopes now share that color |
| 14 | haskell | `languages/haskell/valid/Palette.hs` | **done** | 77 → 10 | `import`/`let`/`in` share `keyword.other.haskell` with the now-correctly-cyan module/where/data/deriving (majority kept); `*`/`**` have no scope at all — the grammar's operator character class omits `*` entirely |
| 15 | markdown | `languages/markdown/valid/palette.md` | **done** | 89 → 4 (+1 known harness artifact on a blockquote continuation line; token counts — the earlier "12 → 3" phrasing counted report *buckets*, inconsistent with every other row) | GFM task-list checkbox `[x]`/`[ ]` brackets carry no scope at all in the Sublime grammar (2); embedded fenced-code Python content is a deliberate non-fix — vim's markdown→python treesitter injection fails silently (no python parser bundled) so vim shows it plain while bat's real grammar injection correctly highlights it, same precedent as HTML's embedded JS (2, one line each) |
| 16a | clojure | `languages/clojure/valid/palette.clj` | **done** | 185 → 28 | call-position symbol scope (`variable.function.clojure`) is shared between ordinary calls (majority, 26, colored) and `let`/`if`/`cond`/`ns`/`->>`/the auto-generated `->Color` record constructor (minority losses, 23 total) — vim treats these as special forms with no distinguishing scope in the grammar; `%`/`&` (anonymous-fn arg placeholder, varargs marker) and record-field-vector names / type-hint symbols after `^` carry no scope at all (5) |
| 16b | elixir | `languages/elixir/valid/palette.ex` | **done** (unblocked 2026-07-07, methodology exception, see below) | 104 → 12 | typespec `::` (in `@spec f() :: type`/`@type t :: ...`) has no scope anywhere in the grammar — searched exhaustively, including the atom-literal regex which lists `::` as an alternative but can't match with only one trailing colon (12→7, largest remaining bucket); `@moduledoc "..."` is parsed as one undifferentiated `comment.documentation.string` region with no internal scope boundary between the attribute name, the quotes, and the string content, but vim colors the three parts differently (Identifier/Delimiter/Comment) — recoloring the shared scope to fix the 2 quote-mark tokens would break the (longer, already-correct) content text, so left alone (3); `_`-prefixed unused-variable naming convention (`_lum`, bare `_`) is a vim-elixir regex-based detector (`elixirUnusedVariable`) with no grammar equivalent, same root cause as lua's ALL_CAPS gap (2) |
| 16c | erlang | `languages/erlang/valid/palette.erl` | **done** | 161 → 0 | fully in sync with vim ground truth |
| 16d | scala | `languages/scala/valid/palette.scala` | **done** | 67 → 5 | interpolated-string quote marks share `punctuation.definition.string.{begin,end}.scala` with plain (non-interpolated) strings, which must stay their current color (2); case-pattern wildcard `_` doesn't match the grammar's `{{varid}}` (requires a trailing ident char after `_`), so it carries no scope in a case pattern (1); Scala 3's colon-triggered trailing-lambda arrow (`foreach: c =>`) reuses the same `storage.type.function.arrow.lambda.scala` scope as ordinary parenthesized lambdas, but this pre-Scala-3 grammar's vim counterpart colors it like a plain keyword instead (1); width-padded `%s` format directives (e.g. `%-8s`) aren't matched by any of the grammar's format-spec patterns — only `%s`/`%#s` with no width digit is supported (1) |

Notes on this row:
- Corpus availability caveat still applies to all four samples (uncommitted
  local files in `~/Developer/srcery-syntax-tester` as of 2026-07-05/06,
  not yet part of the `add-vim-truth-comparison` PR).
- **Neovim's bundled vim-syntax coverage varies per language and must be
  checked before measuring, not assumed from `find`'s exit code alone.**
  A combined `find ... -iname "*a*" -o -iname "*b*"` query silently
  produced wrong/incomplete results here (looked like erlang/scala had no
  syntax files at all); running each language's `find` separately gave
  the correct answer (both are fully supported). Always verify with an
  unambiguous, single-pattern `find` per language, and cross-check
  against an actual `:TOhtml` color dump (`grep -oE
  '\.[A-Za-z0-9_]+\s*\{[^}]*color:[^}]*\}' dump.html`) before concluding
  a filetype has no ground truth — elixir's absence was confirmed this
  way (zero color classes emitted), not inferred from missing files.
- **A scope-selector prefix without the trailing `.<language>` qualifier
  can silently match other languages' scopes with the same prefix.**
  Erlang's io:format control-sequence fix was first written as the bare
  prefix `constant.other.placeholder` (intended to catch all of
  `constant.other.placeholder.{width,precision,...}.erlang` via
  TextMate's dot-segment prefix matching) — but Ruby also emits
  `constant.other.placeholder.ruby` for its own format directives, and
  the unqualified selector matched that too, silently recoloring a Ruby
  format string during the erlang pass (caught by the full regression
  sweep, not the erlang measurement itself). Fixed by spelling out every
  `....erlang`-suffixed sub-scope explicitly instead of relying on an
  unqualified prefix. When a scope needs several co-varying sub-scopes
  matched by prefix, keep the language suffix on every listed variant —
  never drop it for brevity.
- **Elixir methodology exception (2026-07-07, user-approved):** ground
  truth for elixir is measured with `elixir-editors/vim-elixir`
  (https://github.com/elixir-editors/vim-elixir, cloned to a scratch dir
  and appended to `runtimepath` alongside srcery-vim in a one-off copy
  of `dump-vim-html.lua`) instead of vanilla `nvim --clean` alone. This
  is the only row in the queue with a non-uniform ground-truth recipe.
  Justification: srcery-vim's own `colors/srcery.vim` already contains
  dead-code highlight links for `elixirInterpolationDelimiter`,
  `elixirStringDelimiter`, and `elixirDocString` — group names that
  only exist if vim-elixir is installed, confirmed by grepping
  vim-elixir's `syntax/elixir.vim` for those exact names. Without the
  plugin these links are inert and vim renders the whole file as plain
  Normal (which is why the row was originally marked blocked); *with*
  it, srcery-vim's own bespoke Elixir color choices activate. Installing
  vim-elixir isn't a deviation from what srcery-vim's author intended
  for Elixir files — it's necessary to reach it. Anyone reproducing the
  elixir row specifically needs to clone that plugin too; every other
  language in the queue still needs only srcery-vim.

Notes:
- **Corpus availability**: `java`, `clojure`, `elixir`, `erlang`, `scala`
  exist only as *uncommitted* files in the local working copy at
  `~/Developer/srcery-syntax-tester` (as of 2026-07-05). A fresh clone of
  srcery-syntax-tester will not have them. If they're missing, either use
  the local working copy or skip those rows until the corpus PR lands.
- If `bat --list-languages` has no grammar for a language (likely fish),
  mark it "no grammar in bat" and skip.
- **zsh finding (2026, confirmed by measurement, not worth re-litigating):**
  bat renders `.zsh` files through the exact same Bash TextMate grammar as
  `.sh`/`.bash` files (same scope strings), but Neovim dispatches `.zsh`
  files to a *separate* `zsh.vim` syntax file, not `sh.vim`. That
  zsh.vim genuinely disagrees with sh.vim on several *shared* scopes:
  `storage.modifier.shell` (declare/local/readonly/typeset) wants
  Statement-red for bash but Type-blue for zsh; plain variable names on
  the LHS of `=` want Identifier-blue for bash but plain for zsh;
  case-terminator `;;` wants red for bash but white for zsh; even the
  shebang comment wants gray for bash but PreProc-cyan for zsh. Since bat
  cannot tell which dialect produced a given `source.shell` scope, there
  is no selector that satisfies both — any fix for zsh directly regresses
  the already-completed, verified bash alignment. On top of that, zsh-only
  vocabulary (`setopt`, `emulate`, parameter-expansion flags like
  `${(ko)...}`, array slicing `${hex[4,5]}`) has no scope in the
  bash-only grammar at all (ordinary grammar-gap territory, same as other
  languages). Net result: zsh was measured, no theme changes were made,
  and none should be attempted without first deciding whether bash or
  zsh is the priority dialect for the shared scopes above — check with
  the user before changing this call.
- The nvim side needs no extra setup for any of these (vim ships syntax
  files for all); if a dump comes out un-highlighted, check
  `:set filetype?` detection for the sample's extension.

## Definition of done (per language)

- [ ] Every mismatch line is either fixed or classified (gap/quirk) in the
      commit message.
- [ ] All previously-done languages re-measured, counts unchanged.
- [ ] JS operator smoke test passes.
- [ ] Theme lints as valid plist; only palette hexes used; no bare `meta.*`
      selectors added; fontStyle values clean.
- [ ] Queue table updated; committed on `sync-palette-with-srcery-vim`.

When judgment calls arise that contradict an established decision or need a
color not derivable from the vim group table — **stop and ask the user**
rather than inventing policy. Reference how the operator question was
resolved (maintainer intent + measurement, then a split rule).

## Appendix: Java baseline (measured 2026-07-05, before any java fixes)

143 mismatched runs. Raw report, with initial classification hints:

```
bat #FCE8C3 -> vim #917E6B [Delimiter] x42       parens/brackets — vim javaParen* rainbow; QUIRK unless cheap
bat #68A8E4 -> vim #FCE8C3 [Normal]    x26       import paths + type names — broad `entity.name` blue vs vim plain; NEEDS CARE
bat #EF2F27 -> vim #68A8E4 [Type]      x20       int/double/boolean — storage.type red base; add java primitive override like storage.type.c
bat #EF2F27 -> vim #C5B088 [Operator]  x17       `new` — vim javaOperator→Operator white; keyword.operator.new currently red; java-scoped override
bat #FBB829 -> vim #FCE8C3 [Normal]    x5        method names at definition — vim plain, theme yellow via entity.name.function; JUDGMENT (vim java quirk)
bat #FF5C8F -> vim #FCE8C3 [Normal]    x5        ternary/comparison run-majority artifacts; inspect raw render first
bat #C5B088 -> vim #FCE8C3 [Normal]    x5        arithmetic symbols — vim java leaves plain; source.java override like python
bat #2BE4D0 -> vim #917E6B [Comment]   x4        /** */ doc-comment delimiters — java override of doc rule to gray
bat #98BC37 -> vim #FCE8C3 [Normal]    x3        text block .formatted — inspect
bat #2BE4D0 -> vim #C5B088 [vimCommentTitle] x2  doc-comment first sentence — vim white; likely fold into java doc override
bat #EF2F27 -> vim #68A8E4 [StorageClass] x2     class/enum keywords — vim java: StorageClass blue! differs from python/js; java-scoped override
bat #68A8E4 -> vim #FCE8C3 [javaParen1] x2       `[]` — quirk
bat #EF2F27 -> vim #FBB829 [Repeat]    x2        for — like python, source.java loop override
bat #98BC37 -> vim #917E6B [Delimiter] x2        text-block parens — quirk
bat #FCE8C3 -> vim #68A8E4 [StorageClass] x1     record keyword
bat #68A8E4 -> vim #917E6B [Delimiter] x1        [] — quirk
bat #FCE8C3 -> vim #E02C6D [Typedef]   x1        `.class` literal — vim Typedef magenta; scope check
bat #EF2F27 -> vim #C5B088 [Label]     x1        case — vim java Label white (unlike bash red); java override
bat #68A8E4 -> vim #C5B088 [Label]     x1        default
bat #FCE8C3 -> vim #98BC37 [String]    x1        %s format text — inspect
```

Big flags for java specifically: vim java uses **StorageClass (blue) for
`class`/`enum`/`record`** and **Operator (white) for `new`** — opposite of
the red these get from the generic declaration/keyword rules. Both need
java-suffixed overrides, not base-rule changes. The x42 paren Delimiter wall
is vim's rainbow-paren feature; coloring *all* java punctuation gray would
overreach — classify as quirk unless a clean `punctuation.section.*.java`
scope exists and gray matches vim for the common cases.
