# Shiro

A Content-focused theme for Hugo.

## Features

- No JS, just HTML and tiny CSS
- No NPM dependencies

## Installation

Clone the theme into the themes directory, adding it to your project as a Git submodule:

```sh
cd your_hugo_project
git submodule add git@github.com:fuxieorg/hugo-theme-shiro.git themes/shiro
```

Append the following line to `hugo.toml` file, indicating the current theme:

```toml
# ...
theme = "shiro"
# ...
```

Start Hugo’s development server to view the site:

```sh
hugo server
```

## Update

To update the theme, pull the latest changes from the repository by navigating to your Hugo project's root directory and running:

```sh
cd your_hugo_project
git submodule update --remote --merge
```
