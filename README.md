Srcery tmtheme, I created this mainly to get my preferred theme in [bat](https://github.com/sharkdp/bat).

This is untested and in development. Use at own risk.

Generated with [TmTheme Editor](https://tmtheme-editor.herokuapp.com/).

# Usage with bat

Clone srcery-textmate somewhere

```sh
git clone https://github.com/srcery-colors/srcery-textmate ~/my-path
```

Then by following the steps in bat's readme:

```sh
mkdir -p "$(bat --config-dir)/themes"
cd "$(bat --config-dir)/themes"

ln -s ~/my-path/srcery.tmTheme .

# Update the binary cache
bat cache --build
```

Finally, use `bat --list-themes` to check if srcery is available.


# Textastic

![](./assets/screenshots/textastic.jpeg)

The theme works great with [Textastic](<https://www.textasticapp.com>).

Follow the official [guide](<https://www.textasticapp.com/v9/manual/customization/custom_syntax_themes_templates.html>) on how to add the theme.
