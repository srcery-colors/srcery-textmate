<p align="center">
  <img src="https://raw.githubusercontent.com/srcery-colors/srcery-assets/master/src/logo_border.svg">
</p>

<p align="center">
  <a href="https://discord.gg/G6vBMmZ">
    <img alt="Discord" src="https://img.shields.io/discord/714101903377694741?style=for-the-badge&logo=discord&logoColor=%23FCE8C3&label=discord&color=%232C78BF&labelColor=%233A3A3A">
  </a>
  <a href="https://srcery.sh">
    <img alt="Website" src="https://img.shields.io/website?url=https%3A%2F%2Fsrcery.sh&up_color=%23519F50&down_color=%23EF2F27&style=for-the-badge&logo=homepage&logoColor=%23FCE8C3&label=srcery.sh&labelColor=%233A3A3A">
  </a>
  <a href="https://www.npmjs.com/package/@srcery-colors/srcery-palette">
    <img alt="NPM Version" src="https://img.shields.io/npm/v/%40srcery-colors%2Fsrcery-palette?style=for-the-badge&logo=npm&logoColor=%23FCE8C3&label=Palette&color=%23FBB829&labelColor=%233A3A3A">
  </a>
</p>

<h3 align="center">
Srcery TextMate Theme
</h3>

### Description

Srcery TextMate theme. [srcery-vim](https://github.com/srcery-colors/srcery-vim) remains the visual
reference; this repository is the canonical TextMate representation for downstream consumers such as
[srcery-vscode](https://github.com/srcery-colors/srcery-vscode). It can be used anywhere that supports
`tmTheme`.

### Downstream consumers

Use `srcery.tmTheme` as the source for any TextMate-based integration. A downstream consumer should
vendor or synchronize the file rather than duplicate its token rules, so fixes and deliberate grammar
workarounds land consistently. Consumers with a different scope model, such as tree-sitter-based themes,
should map the same visual intent rather than copy TextMate selectors verbatim.

#### Preview
##### Javascript
![javascript](https://raw.githubusercontent.com/srcery-colors/srcery-assets/refs/heads/master/textmate/javascript.png)

##### CSS
![css](https://raw.githubusercontent.com/srcery-colors/srcery-assets/refs/heads/master/textmate/css.png)

### Usage with bat

[bat](https://github.com/sharkdp/bat) finds themes anywhere inside its
`themes` folder, so you can clone this repo straight into it:

```bash
git clone https://github.com/srcery-colors/srcery-textmate "$(bat --config-dir)/themes/srcery"
bat cache --build
```

Try it:

```bash
bat --theme=srcery README.md
```

To make it the default, add `--theme=srcery` to your bat config file
(`bat --config-file` shows where it is), or set `BAT_THEME=srcery` in your
shell.

To update later:

```bash
git -C "$(bat --config-dir)/themes/srcery" pull
bat cache --build
```


### Textastic

![textastic preview](https://raw.githubusercontent.com/srcery-colors/srcery-assets/refs/heads/master/textmate/textastic.jpeg)

The theme works great with [Textastic](<https://www.textasticapp.com>).

Follow the official [guide](<https://www.textasticapp.com/v9/manual/customization/custom_syntax_themes_templates.html>) on how to add the theme.

### Theme Editor

Created with [TmTheme Editor](https://tmtheme-editor.herokuapp.com/).

### Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

### License

[MIT](LICENSE)
