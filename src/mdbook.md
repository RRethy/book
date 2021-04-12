# mdBook

I've started using [mdBook](https://github.com/rust-lang/mdBook) as a way to get better at writing and force myself to have a better understanding of what I'm working on. I'm hoping that by writing everything down, I'll force myself to get a better understanding of the topic and give my future self resources to use when revisiting the topic.

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
