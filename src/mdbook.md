# About this Book

I've started using [mdBook](https://github.com/rust-lang/mdBook) as a way to get better at writing and force myself to have a better understanding of what I'm working on. Hopefully this will help my future self when I need a reference, or potentially someone else.

In terms of mdBook itself, the best resource I've found is the guide book built with mdBook itself found at https://rust-lang.github.io/mdBook.

## Quickstart

Create a book with

```sh
cargo install mdbook
mkdir book
cd book
mdbook init
```

The structure is found in `src/SUMMARY.md` and supports limited markdown syntax:

```markdown
# This title is ignored
[Non-numered top-level section](path.md)
# Unclickable top-level title
- [Numered chapter title](path2.md)
    - [Nested sub-chapters](path3.md)
[Non-numered top-level section](path.md)
```

The contents files support normal markdown.

## Deploy to GitHub Pages

Add a `.github/workflows/gh-pages.yml` file with the contents:

```yaml
name: github pages

on:
  push:
    branches:
      - master

jobs:
  deploy:
    runs-on: ubuntu-18.04
    steps:
      - uses: actions/checkout@v2

      - name: Setup mdBook
        uses: peaceiris/actions-mdbook@v1
        with:
          mdbook-version: 'latest'

      - run: mdbook build

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./book
```

Go to `https://github.com/{username}/{repo}/settings/pages` and ensure the source branch is `gh-pages`.

I had to wait some time for it to deploy.
