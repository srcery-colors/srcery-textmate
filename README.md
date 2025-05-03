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

Srcery TextMate theme, best effort to match [srcery-vim](https://github.com/srcery-colors/srcery-vim) colors. Can be used
anywhere that supports `tmTheme`.

#### Preview
##### Javascript
![javascript](https://raw.githubusercontent.com/srcery-colors/srcery-assets/refs/heads/master/textmate/javascript.png)

##### CSS
![css](https://raw.githubusercontent.com/srcery-colors/srcery-assets/refs/heads/master/textmate/css.png)

### Usage with bat

- [sharkdp/bat: A cat(1) clone with wings.](https://github.com/sharkdp/bat)

Clone srcery-textmate somewhere

```bash
git clone https://github.com/srcery-colors/srcery-textmate ~/my-path
```

Then by following the steps in bat's readme:

```bash
mkdir -p "$(bat --config-dir)/themes"
cd "$(bat --config-dir)/themes"

ln -s ~/my-path/srcery.tmTheme .

bat cache --build # Update the binary cache
```

Finally, use `bat --list-themes` to check if srcery is available.


### Textastic

![textastic preview](https://raw.githubusercontent.com/srcery-colors/srcery-assets/refs/heads/master/textmate/textastic.jpeg)

The theme works great with [Textastic](<https://www.textasticapp.com>).

Follow the official [guide](<https://www.textasticapp.com/v9/manual/customization/custom_syntax_themes_templates.html>) on how to add the theme.

### Theme Editor

Created with [TmTheme Editor](https://tmtheme-editor.herokuapp.com/).

### License

[MIT](LICENSE)
