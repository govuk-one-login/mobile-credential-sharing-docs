# mobile-credential-sharing-docs

## mDL Solution Architecture

This repository contains the canonical architecture specifications and system
component logic for the mDL Verifier and Holder apps.

The app repositories plan to include this repository as a **Git Submodule**.

### Structure

* **[Verifier Architecture]**: The complete
  user flow and logic for the Verifier.
* **[Holder Architecture]**: The complete user
  flow and logic for the Holder.
* **[Components]**: Shared system components used by both sides.
  For example:
  * Bluetooth
  * Cryptographic features
  * Finite State Machine (FSM)

### Integration

When implementing features, refer to the specs here. If the spec changes, update
the submodule in the app repository to pull the latest version.

### Updating Specs

1. Make changes in this repository (PRs are welcome).
2. Merge to `main`.
3. Go to the app repository and run `git submodule update --remote`.

## Developer set up

Please see the [Developer documentation] for information regarding set up and
validation of the repository.

<!-- MARK: References -->
[Components]: ./components/
[Developer documentation]: ./docs/dev/README.md
[Holder Architecture]: ./holder-solution-architecture.md
[Verifier Architecture]: ./verifier-solution-architecture.md
