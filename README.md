# mobile-credential-sharing-docs

## mDL Solution Architecture

This repository contains the canonical architecture specifications and system component logic for the mDL Verifier and Holder applications.

It is designed to be included as a **Git Submodule** in the app repositories.

### Structure

* **[Verifier Architecture](verifier-solution-architecture.md)**: The complete user flow and logic for the Verifier.
* **[Holder Architecture](holder-solution-architecture.md)**: The complete user flow and logic for the Holder.
* **[Components/](components/)**: Shared system components used by both sides (eg. Bluetooth, Crypto, Session FSM).

### Integration

When implementing features, refer to the specs here. If the spec changes, update the submodule in the app repo to pull the latest version.

### Updating Specs
1. Make changes in this repository (PRs welcome.
2. Merge to `main`.
3. Go to the app repo and run `git submodule update --remote`.
