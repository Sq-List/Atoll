# Personal Atoll App Build

## Goal

Produce a downloadable `Atoll.app.zip` containing the static plugin host so the feature can be used before the upstream pull request is merged.

The personal build must not modify the upstream pull request, the fork's `dev` branch, or an existing app under `/Applications`.

## Branch and trigger

The build lives on `codex/static-plugin-app-build`, based on `feat/static-web-plugins`. A workflow committed only to this branch runs when the branch is pushed to the fork.

This keeps personal packaging separate from the contribution while allowing a new build to be requested by pushing an updated branch.

## Build

The workflow uses a GitHub-hosted macOS runner and the latest stable Xcode. It follows the repository CI prerequisites, builds the `DynamicIsland` scheme in Release configuration, and writes build products to a known temporary directory.

The build uses ad-hoc signing because the fork does not have the maintainer's Developer ID, Sparkle, or notarization credentials. The workflow verifies the resulting app with `codesign` before packaging it.

## Artifact

The workflow locates exactly one `Atoll.app`, archives it with `ditto` as `Atoll-static-plugin-host.zip`, and uploads the archive as a GitHub Actions artifact with seven-day retention.

The completed artifact is downloaded to `/Users/sqlist/Project/other/AtollBuild`. It is not copied to `/Applications` automatically.

## Failure handling

The workflow fails if dependency preparation or compilation fails, if `Atoll.app` is missing, or if signature verification fails. Build diagnostics are uploaded when available. Packaging does not hide or ignore these failures.

## Validation

Completion requires:

- the fork workflow succeeds;
- the artifact can be downloaded and unzipped locally;
- the app contains an executable and passes `codesign --verify --deep --strict`;
- the archive contains the static plugin host files from the feature branch;
- the upstream pull request remains unchanged by the packaging workflow.

Running or installing the app is a separate user-approved action because it may replace an existing installation or request macOS permissions.
