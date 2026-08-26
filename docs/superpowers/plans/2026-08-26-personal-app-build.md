# Personal Atoll App Build Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the static-plugin version of Atoll in the fork and download a verified `Atoll.app.zip` without changing the upstream pull request or installing the app.

**Architecture:** A workflow exists only on `codex/static-plugin-app-build` and runs when that branch is pushed. It follows the repository CI preparation, produces an ad-hoc-signed Release app in a known DerivedData directory, verifies it, archives it with `ditto`, and uploads the ZIP for seven days.

**Tech Stack:** GitHub Actions, Xcode, `xcodebuild`, macOS `codesign`, `ditto`, GitHub CLI.

---

### Task 1: Add the fork-only app packaging workflow

**Files:**
- Create: `.github/workflows/personal-static-plugin-app.yml`

- [ ] **Step 1: Confirm the workflow is absent**

Run:

```bash
test ! -e .github/workflows/personal-static-plugin-app.yml
```

Expected: exit status 0.

- [ ] **Step 2: Create the workflow**

Create `.github/workflows/personal-static-plugin-app.yml` with exactly:

```yaml
name: Personal Static Plugin App

on:
  push:
    branches:
      - codex/static-plugin-app-build

permissions:
  contents: read

concurrency:
  group: personal-static-plugin-app-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    name: Build Atoll.app
    runs-on: macos-15

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4
        with:
          persist-credentials: false

      - name: Select Xcode Version
        uses: maxim-lobanov/setup-xcode@v1
        with:
          xcode-version: latest-stable

      - name: Validate repository configuration
        run: python3 -m unittest tests.test_privacy_configuration tests.test_timer_lifecycle

      - name: Prepare build environment
        run: |
          sudo xcodebuild -runFirstLaunch || true
          sudo DevToolsSecurity -enable

          if ! plutil_out=$(plutil -remove com.apple.security.mach-services DynamicIsland/DynamicIsland.entitlements 2>&1); then
            if printf '%s' "$plutil_out" | grep -q "No value to remove"; then
              echo "mach-services entitlement already absent, continuing."
            else
              echo "::error::plutil failed to modify entitlements: $plutil_out"
              exit 1
            fi
          fi

      - name: Build Release app
        run: |
          xcodebuild build \
            -project DynamicIsland.xcodeproj \
            -scheme DynamicIsland \
            -configuration Release \
            -destination "platform=macOS" \
            -derivedDataPath "$RUNNER_TEMP/DerivedData" \
            -resultBundlePath "$RUNNER_TEMP/BuildResults.xcresult" \
            CODE_SIGN_IDENTITY="-" \
            CODE_SIGNING_REQUIRED=NO \
            CODE_SIGNING_ALLOWED=YES \
            CODE_SIGN_INJECT_BASE_ENTITLEMENTS=YES

      - name: Verify and package app
        run: |
          APP_PATH="$RUNNER_TEMP/DerivedData/Build/Products/Release/Atoll.app"
          ARCHIVE_PATH="$RUNNER_TEMP/Atoll-static-plugin-host.zip"

          test -d "$APP_PATH"
          test -x "$APP_PATH/Contents/MacOS/Atoll"
          codesign --verify --deep --strict --verbose=2 "$APP_PATH"
          ditto -c -k --sequesterRsrc --keepParent "$APP_PATH" "$ARCHIVE_PATH"
          test -s "$ARCHIVE_PATH"

      - name: Surface failure diagnostics
        if: failure()
        run: |
          xcrun xcresulttool get build-results --path "$RUNNER_TEMP/BuildResults.xcresult" 2>/dev/null \
            || xcrun xcresulttool get --path "$RUNNER_TEMP/BuildResults.xcresult" --format json 2>/dev/null | head -c 20000 \
            || echo "No xcresult available."

      - name: Upload app
        uses: actions/upload-artifact@v4
        with:
          name: Atoll-static-plugin-host
          path: ${{ runner.temp }}/Atoll-static-plugin-host.zip
          if-no-files-found: error
          retention-days: 7

      - name: Upload failure results
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: personal-build-results
          path: ${{ runner.temp }}/BuildResults.xcresult
          if-no-files-found: ignore
          retention-days: 7
```

- [ ] **Step 3: Validate syntax and diff cleanliness**

Run:

```bash
ruby -e 'require "yaml"; value = YAML.load_file(".github/workflows/personal-static-plugin-app.yml"); abort unless value.is_a?(Hash)'
git diff --check
```

Expected: both commands exit 0 with no output.

- [ ] **Step 4: Commit the workflow**

```bash
git add .github/workflows/personal-static-plugin-app.yml
git commit -m "ci: package personal static plugin build"
```

Expected: one workflow file is committed; the other untracked `docs/` files remain uncommitted.

### Task 2: Run the fork build

**Files:**
- No local file changes.

- [ ] **Step 1: Push only the personal build branch**

Run through the configured network proxy:

```bash
git push -u origin codex/static-plugin-app-build
```

Expected: the new branch appears in `Sq-List/Atoll` and the push triggers `Personal Static Plugin App`.

- [ ] **Step 2: Resolve the workflow run ID**

```bash
gh run list --repo Sq-List/Atoll --branch codex/static-plugin-app-build --workflow personal-static-plugin-app.yml --limit 1 --json databaseId,status,conclusion,url,headSha
```

Expected: one run whose `headSha` matches local `HEAD`.

- [ ] **Step 3: Wait for the run**

```bash
gh run watch RUN_ID --repo Sq-List/Atoll --exit-status --interval 10
```

Expected: `Build Atoll.app` completes successfully. On failure, inspect only this run with `gh run view RUN_ID --repo Sq-List/Atoll --log-failed`, correct the concrete workflow or build error, and repeat the focused checks before pushing.

### Task 3: Download and verify the app locally

**Files:**
- Create: `/Users/sqlist/Project/other/AtollBuild/Atoll-static-plugin-host.zip`
- Create: `/Users/sqlist/Project/other/AtollBuild/extracted/Atoll.app`

- [ ] **Step 1: Create the destination without touching an existing app installation**

```bash
mkdir -p /Users/sqlist/Project/other/AtollBuild/extracted
```

Expected: the destination exists; `/Applications/Atoll.app` is unchanged.

- [ ] **Step 2: Download the named artifact**

```bash
gh run download RUN_ID --repo Sq-List/Atoll --name Atoll-static-plugin-host --dir /Users/sqlist/Project/other/AtollBuild
```

Expected: `/Users/sqlist/Project/other/AtollBuild/Atoll-static-plugin-host.zip` exists and is non-empty.

- [ ] **Step 3: Extract and verify**

```bash
ditto -x -k /Users/sqlist/Project/other/AtollBuild/Atoll-static-plugin-host.zip /Users/sqlist/Project/other/AtollBuild/extracted
test -x /Users/sqlist/Project/other/AtollBuild/extracted/Atoll.app/Contents/MacOS/Atoll
codesign --verify --deep --strict --verbose=2 /Users/sqlist/Project/other/AtollBuild/extracted/Atoll.app
strings /Users/sqlist/Project/other/AtollBuild/extracted/Atoll.app/Contents/MacOS/Atoll | rg -q 'A plugin with ID .* is already installed'
```

Expected: all commands exit 0. The final check proves the downloaded binary contains the static-plugin replacement path.

### Task 4: Confirm contribution isolation

**Files:**
- No file changes.

- [ ] **Step 1: Verify branch contents and upstream PR head**

```bash
git status --short --branch
gh pr view 781 --repo Ebullioscopic/Atoll --json headRefOid,isDraft,state,url
```

Expected: the personal branch tracks its fork branch; only pre-existing unrelated `docs/` files may remain untracked. Upstream PR #781 remains ready for review at head `f787281b96352dc768ae91ba7452a87a0b535c58`.

- [ ] **Step 2: Report the artifact path without installing it**

Report `/Users/sqlist/Project/other/AtollBuild/Atoll-static-plugin-host.zip` and `/Users/sqlist/Project/other/AtollBuild/extracted/Atoll.app`. Do not copy either into `/Applications` until the user explicitly approves replacement or installation.
