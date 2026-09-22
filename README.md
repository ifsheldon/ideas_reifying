# Ideas Reifying

This is the source code of the blogsite [ideas reifying](https://ideas.reify.ing).

## Local development

Use Zola 0.23.6 or newer; this site is tested with 0.23.6.
From the repository root, initialize the theme submodule and build the site:

```sh
git submodule update --init --recursive
zola build
```

To preview the site locally with automatic rebuilding:

```sh
zola serve
```

The theme uses Tera 2 components; see the [theme documentation](themes/float/README.en.md#template-customization) when changing template overrides.

## License

Unless explicitly specified, the license of the blog content is [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/).
