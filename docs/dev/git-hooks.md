# Git hooks

The code base utilises various git hooks that exist within the
[Git hooks directory]. This helps to ensure that documentation quality meets
expectations before being permanently added into the repository.

## Installation

### Via git configuration

Git configuration can override the `core.hooksPath` property to point to a
non-default directory:

```shell
git config core.hooksPath $(git rev-parse --show-toplevel)/.config/githooks
```

[Git hooks directory]: /.config/githooks
