# Zed Docs

Welcome to Zed's documentation.

This is built on push to `main` and published automatically to [https://zed.dev/docs](https://zed.dev/docs).

To preview the docs locally you will need to install [mdBook](https://rust-lang.github.io/mdBook/) (`cargo install mdbook@0.4.40`) and then run:

```sh
mdbook serve docs
```

It's important to note the version number above. For an unknown reason, as of 2025-04-23, running 0.4.48 will cause odd URL behavior that breaks docs.

Before committing, verify that the docs are formatted in the way Prettier expects with:

```
cd docs && pnpm dlx prettier@3.5.0 . --write && cd ..
```

## Preprocessor

We have a custom mdbook preprocessor for interfacing with our crates (`crates/docs_preprocessor`).

If for some reason you need to bypass the docs preprocessor, you can comment out `[preprocessor.zed_docs_preprocessor]
` from the `book.toml`.:

## Images and videos

To add images or videos to the docs, upload them to another location (e.g., zed.dev, GitHub's asset storage) and then link out to them from the docs.

Putting binary assets such as images in the Git repository will bloat the repository size over time.

## Internal notes:

- We have a Cloudflare router called `docs-proxy` that intercepts requests to `zed.dev/docs` and forwards them to the "docs" Cloudflare Pages project.
- CI uploads a new version to the Pages project from `.github/workflows/deploy_docs.yml` on every push to `main`.

### Table of Contents

The table of contents files (`theme/page-toc.js` and `theme/page-doc.css`) were initially generated by [`mdbook-pagetoc`](https://crates.io/crates/mdbook-pagetoc).

Since all this preprocessor does is generate the static assets, we don't need to keep it around once they have been generated.

## Referencing Keybindings and Actions

When referencing keybindings or actions, use the following formats:

### Keybindings:

`{#kb scope::Action}` - e.g., `{#kb zed::OpenSettings}`.

This will output a code element like: `<code>Cmd+,|Ctrl+,</code>`. We then use a client-side plugin to show the actual keybinding based on the user's platform.

By using the action name, we can ensure that the keybinding is always up-to-date rather than hardcoding the keybinding.

### Actions:

`{#action scope::Action}` - e.g., `{#action zed::OpenSettings}`.

This will render a human-readable version of the action name, e.g., "zed: open settings", and will allow us to implement things like additional context on hover, etc.

### Creating New Templates

Templates are just functions that modify the source of the docs pages (usually with a regex match & replace). You can see how the actions and keybindings are templated in `crates/docs_preprocessor/src/main.rs` for reference on how to create new templates.

### References

- Template Trait: crates/docs_preprocessor/src/templates.rs
- Example template: crates/docs_preprocessor/src/templates/keybinding.rs
- Client-side plugins: docs/theme/plugins.js

## Postprocessor

A postprocessor is implemented as a sub-command of `docs_preprocessor` that wraps the builtin `html` renderer and applies post-processing to the `html` files, to add support for page-specific title and meta description values.

An example of the syntax can be found in `git.md`, as well as below

```md
---
title: Some more detailed title for this page
description: A page-specific description
---

# Editor
```

The above will be transformed into (with non-relevant tags removed)

```html
<head>
  <title>Editor | Some more detailed title for this page</title>
  <meta name="description" contents="A page-specific description" />
</head>
<body>
  <h1>Editor</h1>
</body>
```

If no front-matter is provided, or If one or both keys aren't provided, the title and description will be set based on the `default-title` and `default-description` keys in `book.toml` respectively.

### Implementation details

Unfortunately, `mdbook` does not support post-processing like it does pre-processing, and only supports defining one description to put in the meta tag per book rather than per file. So in order to apply post-processing (necessary to modify the html head tags) the global book description is set to a marker value `#description#` and the html renderer is replaced with a sub-command of `docs_preprocessor` that wraps the builtin `html` renderer and applies post-processing to the `html` files, replacing the marker value and the `<title>(.*)</title>` with the contents of the front-matter if there is one.

### Known limitations

The front-matter parsing is extremely simple, which avoids needing to take on an additional dependency, or implement full yaml parsing.

- Double quotes and multi-line values are not supported, i.e. Keys and values must be entirely on the same line, with no double quotes around the value.

The following will not work:

```md
---
title: Some
  Multi-line
  Title
---
```

And neither will:

```md
---
title: "Some title"
---
```

- The front-matter must be at the top of the file, with only white-space preceding it

- The contents of the title and description will not be html-escaped. They should be simple ascii text with no unicode or emoji characters
