# Developer Tooling

## Overview

The [Repository Brewfile] contains all of the necessary dependencies so that
developers using a freshly cloned repository can then perform the proceeding
command:

```shell
brew bundle check || brew bundle install
```

- [Homebrew]

  An agnostic package manager for downloading additional repository
  dependencies.

- [Shellcheck linter]

  A code analysis tool that maintains quality for the
  [Repository shell scripts]. The [Repository git hooks] run this tool during
  the 'pre-commit' stage. However, developers can manually run `shellcheck`
  linting with the proceeding script:

  ```shell
  scripts/lint/shellcheck/run
  ```

- [Vale prose linter]

  Maintains the literary quality of the documentation. The
  [Repository git hooks] run this tool during the 'pre-commit' stage. However,
  developers can manually run `shellcheck` linting with the proceeding script:

  ```shell
  scripts/lint/vale/run
  ```

- [Visual Studio Code] (VSCode)

  The primary Integrated Development Environment (IDE) used within this
  repository, providing [Workspace configurations] and [Recommended extensions]
  for the developer's ease of use.

<!-- MARK: References -->
[Homebrew]: https://brew.sh/
[Recommended extensions]: /.vscode/extensions.json
[Repository Brewfile]: /Brewfile
[Repository git hooks]: /.config/githooks/
[Repository shell scripts]: /scripts/
[Shellcheck linter]: https://www.shellcheck.net/
[Visual Studio Code]: https://code.visualstudio.com/
[Vale prose linter]: https://vale.sh/
[Workspace configurations]: /.vscode/settings.json
