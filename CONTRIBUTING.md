# Contributing

`srcery.tmTheme` is the canonical TextMate source for Srcery.
[srcery-vim](https://github.com/srcery-colors/srcery-vim) is the visual
reference: when the two disagree, vim wins. Downstream consumers such as
srcery-vscode sync from this file, so a change here reaches them.

## Before you change the theme

Read [docs/aligning-with-vim.md](docs/aligning-with-vim.md). It has the
measuring workflow, the selector rules and the decisions that are settled.

Three rules that cause most review comments:

- **Palette only.** Every hex must come from
  [`@srcery-colors/srcery-palette`](https://github.com/srcery-colors/srcery-palette)
  at the version pinned in the guide.
- **Measure, don't eyeball.** Back each change with a before/after count from
  the harness.
- **Don't fix one language with a generic rule.** Keep the language suffix on
  every selector, and re-check all other languages.

## Commits and pull requests

- Use [Conventional Commits](https://www.conventionalcommits.org/):
  `fix(<lang>): align rendering with vim output`, `docs: …`, `chore: …`.
- One language per commit. Put `Mismatched runs: <before> -> <after>` in the
  body.
- Keep a pull request to one concern, and describe what you measured.
- For a docs-only change, say so; no measurement is needed.

## Reporting a problem

Open an issue with the language, a short code sample, the color you see and the
color you expected (or what vim shows).

## License

By contributing you agree that your work is released under the
[MIT license](LICENSE).
