# Static Plugin Runtime Fixes

## Goal

Fix two defects observed in the Developer Tools static plugin:

- local JavaScript does not execute in Atoll's file-backed WKWebView, leaving all tabs inert;
- the notch host clamps plugin content to 332 points, so the requested content height is truncated.

## JavaScript loading

The plugin will load ordered classic scripts instead of an ES module entrypoint. `conversions.js` will expose its tested conversion functions through `globalThis.AtollConversions`, and `app.js` will consume that object after `conversions.js` has loaded.

This keeps the runtime offline and file-backed while avoiding WKWebView's local ES-module loading restriction. Tests will execute the same `conversions.js` file in a Node VM and verify the HTML uses classic scripts in the required order. A WKWebView probe will verify that the timestamp initialises and a Base64 tab click changes the visible panel.

The plugin version will become `1.0.1`, allowing the existing same-ID Replace flow to communicate that a newer package is being installed.

## Plugin height

Atoll will move static-plugin height resolution into an internal, testable helper. The result will:

- never be shorter than Atoll's normal open-notch height;
- respect the plugin's requested height when it fits;
- never exceed 70 percent of the current screen's visible height.

If the screen height is unavailable, the existing one-extra-row limit remains the fallback. Extension-app sizing is unchanged.

Developer Tools will request 460 points. Content that still exceeds the available space continues to scroll inside the plugin.

## Branches and delivery

The host fix and its tests will be committed to `feat/static-web-plugins`, updating upstream PR #781. The plugin files remain in `/Users/sqlist/Project/other/AtollDeveloperTools` and are not added to the Atoll contribution.

After focused tests pass, the host-fix commit will also be applied to `codex/static-plugin-app-build`. Pushing that branch will generate a replacement personal `Atoll.app` artifact.

## Validation

Completion requires:

- the pre-fix WKWebView probe reproduces an empty timestamp and inert Base64 tab;
- the post-fix probe shows a populated timestamp and the Base64 panel visible after clicking;
- Node tests cover conversions and classic-script ordering;
- Swift tests cover minimum, requested, and screen-clamped plugin heights;
- Atoll's existing regression tests pass;
- macOS 15 and macOS 26 CI builds pass on the updated contribution branch;
- the personal Release app builds and is downloadable;
- replacing the installed plugin with version `1.0.1` is the only manual step left to the user.
