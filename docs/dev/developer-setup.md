# Developer set up

1. Install the third party tools described within the [Developer tooling]
   document.

2. Clone the repository if you've yet to do so:

   ```shell
   git clone https://github.com/govuk-one-login/mobile-credential-sharing-docs.git
   ```

3. Configure the git repository.

   - Configure commit signing:

     This repository enforces signed commits with GNU Privacy Guard (GPG)
     keys.  
     Therefore, [generate a new GPG key] if you've yet to do so.  
     Add the key to your [GitHub key settings].  
     [Inform git about your signing key] and enable the `commit.gpgsign` git
     configuration property.

   - Configure the [Repository git hooks].

4. Open the git repository's root directory within [Visual Studio Code]
   (VSCode).

5. Install the [Workspace recommended extensions].

   - via Notification:

     When opening the repository within VSCode, the Integrated Development
     Environment (IDE) automatically checks the availability of all recommended
     extensions. If there's missing extensions, a notification request
     (usually at the bottom right of the VSCode window) appears, prompting the
     developer to install what's missing.

   - via the Extensions menu:

     After opening VSCode's extensions menu (commonly found on the far left
     ribbon menu), searching for `@recommended` provides a list of workspace
     recommendations for manual installation.

<!-- MARK: References -->
[Developer tooling]: ./developer-tooling.md
[generate a new GPG key]:
    https://docs.github.com/en/authentication/managing-commit-signature-verification/generating-a-new-gpg-key
[GitHub key settings]: https://github.com/settings/keys
[Inform git about your signing key]:
    https://docs.github.com/en/authentication/managing-commit-signature-verification/telling-git-about-your-signing-key
[Repository git hooks]: ./git-hooks.md
[Visual Studio Code]: https://code.visualstudio.com/
[Workspace recommended extensions]: /.vscode/extensions.json
