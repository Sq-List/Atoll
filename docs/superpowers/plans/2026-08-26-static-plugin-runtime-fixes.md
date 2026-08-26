# Static Plugin Runtime Fixes Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make local static-plugin JavaScript interactive in WKWebView and let plugins request a useful notch height within 70 percent of the visible screen.

**Architecture:** Developer Tools uses ordered classic scripts so the same file-backed resources work in WKWebView without a local ES-module origin. Atoll resolves plugin height through a small pure helper that ContentView calls with the current screen height and the existing 332-point fallback.

**Tech Stack:** Swift, SwiftUI, WKWebView, XCTest, JavaScript, Node test runner.

---

### Task 1: Add failing Developer Tools runtime tests

**Files:**
- Create: `/Users/sqlist/Project/other/AtollDeveloperTools/tests/wkwebview-probe.swift`
- Modify: `/Users/sqlist/Project/other/AtollDeveloperTools/tests/conversions.test.js`

- [ ] **Step 1: Make the conversion tests execute the runtime script as a classic script**

Replace the ES-module import in `tests/conversions.test.js` with a Node VM loader:

```js
import test from "node:test";
import assert from "node:assert/strict";
import { readFileSync } from "node:fs";
import vm from "node:vm";

const conversionsURL = new URL("../DeveloperTools.atollplugin/assets/conversions.js", import.meta.url);
const context = vm.createContext({ TextEncoder, TextDecoder, atob, btoa });
vm.runInContext(readFileSync(conversionsURL, "utf8"), context, { filename: conversionsURL.pathname });

const {
  bd09ToGcj02,
  decodeBase64,
  encodeBase64,
  gcj02ToBd09,
  gcj02ToWgs84,
  normalizeUnixTimestamp,
  wgs84ToGcj02,
} = context.AtollConversions;
```

Keep the five existing behavior tests, convert the outside-China array assertion to `Array.from(...)`, and add:

```js
test("loads classic scripts in dependency order", () => {
  const html = readFileSync(
    new URL("../DeveloperTools.atollplugin/index.html", import.meta.url),
    "utf8",
  );
  const conversionsIndex = html.indexOf('<script src="assets/conversions.js"></script>');
  const appIndex = html.indexOf('<script src="assets/app.js"></script>');

  assert.ok(conversionsIndex >= 0);
  assert.ok(appIndex > conversionsIndex);
  assert.doesNotMatch(html, /type="module"/);
});
```

- [ ] **Step 2: Add a WKWebView regression probe**

Create `tests/wkwebview-probe.swift`:

```swift
import AppKit
import Foundation
import WebKit

struct PageState: Decodable {
    let timestamp: String
    let timestampHidden: Bool
    let base64Hidden: Bool
}

private let stateScript = """
JSON.stringify({
  timestamp: document.querySelector('#timestamp-input')?.value ?? '',
  timestampHidden: document.querySelector('#timestamp')?.hidden ?? true,
  base64Hidden: document.querySelector('#base64')?.hidden ?? true
})
"""

private func decodeState(_ value: Any?) throws -> PageState {
    guard let json = value as? String else {
        throw NSError(domain: "WKWebViewProbe", code: 1)
    }
    return try JSONDecoder().decode(PageState.self, from: Data(json.utf8))
}

final class ProbeDelegate: NSObject, WKNavigationDelegate {
    func webView(_ webView: WKWebView, didFinish navigation: WKNavigation!) {
        webView.evaluateJavaScript(stateScript) { value, error in
            do {
                if let error { throw error }
                let initial = try decodeState(value)
                print("initial=\(initial)")
                guard !initial.timestamp.isEmpty else { exit(1) }

                webView.evaluateJavaScript("document.querySelector('[data-panel=base64]').click()") { _, clickError in
                    if let clickError {
                        print("clickError=\(clickError)")
                        exit(1)
                    }
                    webView.evaluateJavaScript(stateScript) { clickedValue, clickedError in
                        do {
                            if let clickedError { throw clickedError }
                            let clicked = try decodeState(clickedValue)
                            print("afterClick=\(clicked)")
                            exit(clicked.timestampHidden && !clicked.base64Hidden ? 0 : 1)
                        } catch {
                            print("afterClickError=\(error)")
                            exit(1)
                        }
                    }
                }
            } catch {
                print("initialError=\(error)")
                exit(1)
            }
        }
    }

    func webView(_ webView: WKWebView, didFail navigation: WKNavigation!, withError error: Error) {
        print("navigationError=\(error)")
        exit(1)
    }
}

let projectURL = URL(fileURLWithPath: #filePath)
    .deletingLastPathComponent()
    .deletingLastPathComponent()
let pluginURL = projectURL.appendingPathComponent("DeveloperTools.atollplugin", isDirectory: true)
let webView = WKWebView(frame: NSRect(x: 0, y: 0, width: 1462, height: 332))
let delegate = ProbeDelegate()
webView.navigationDelegate = delegate
webView.loadFileURL(
    pluginURL.appendingPathComponent("index.html"),
    allowingReadAccessTo: pluginURL
)

DispatchQueue.main.asyncAfter(deadline: .now() + 10) {
    print("timeout")
    exit(1)
}
RunLoop.main.run()
```

- [ ] **Step 3: Verify RED**

Run:

```bash
cd /Users/sqlist/Project/other/AtollDeveloperTools
npm test
DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer xcrun swift tests/wkwebview-probe.swift
```

Expected: Node fails on the current `export` syntax or missing classic scripts, and the WKWebView probe fails because the timestamp remains empty and Base64 remains hidden.

### Task 2: Make Developer Tools file-backed scripts executable

**Files:**
- Modify: `/Users/sqlist/Project/other/AtollDeveloperTools/DeveloperTools.atollplugin/assets/conversions.js`
- Modify: `/Users/sqlist/Project/other/AtollDeveloperTools/DeveloperTools.atollplugin/assets/app.js`
- Modify: `/Users/sqlist/Project/other/AtollDeveloperTools/DeveloperTools.atollplugin/index.html`
- Modify: `/Users/sqlist/Project/other/AtollDeveloperTools/DeveloperTools.atollplugin/manifest.json`

- [ ] **Step 1: Expose conversion functions through one global object**

Remove all `export` keywords from `conversions.js`, retain the existing functions unchanged, and append:

```js
globalThis.AtollConversions = Object.freeze({
  bd09ToGcj02,
  decodeBase64,
  encodeBase64,
  gcj02ToBd09,
  gcj02ToWgs84,
  normalizeUnixTimestamp,
  wgs84ToGcj02,
});
```

- [ ] **Step 2: Consume the global object in app.js**

Replace the ES-module import with:

```js
const {
  bd09ToGcj02,
  decodeBase64,
  encodeBase64,
  gcj02ToBd09,
  gcj02ToWgs84,
  normalizeUnixTimestamp,
  wgs84ToGcj02,
} = globalThis.AtollConversions;
```

- [ ] **Step 3: Load scripts in order and increase the requested height**

Replace the module script in `index.html` with:

```html
<script src="assets/conversions.js"></script>
<script src="assets/app.js"></script>
```

Set `version` to `1.0.1` and `tab.preferredHeight` to `460` in `manifest.json`.

- [ ] **Step 4: Verify GREEN**

Run:

```bash
cd /Users/sqlist/Project/other/AtollDeveloperTools
npm test
DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer xcrun swift tests/wkwebview-probe.swift
```

Expected: six Node tests pass; the WKWebView output has a non-empty timestamp initially and `base64Hidden: false` after the click.

### Task 3: Add a failing Atoll height test

**Files:**
- Modify: `DynamicIslandTests/StaticPluginPackageValidatorTests.swift`

- [ ] **Step 1: Switch to the contribution branch**

```bash
git checkout feat/static-web-plugins
```

Expected: the three pre-existing untracked design/plan files remain untouched.

- [ ] **Step 2: Add layout tests before production code**

Append:

```swift
final class StaticPluginLayoutTests: XCTestCase {
    func testUsesRequestedHeightWhenItFitsScreen() {
        XCTAssertEqual(
            StaticPluginLayout.resolvedHeight(
                preferredHeight: 460,
                baseHeight: 200,
                visibleScreenHeight: 900,
                fallbackMaximumHeight: 332
            ),
            460
        )
    }

    func testNeverShrinksBelowBaseHeight() {
        XCTAssertEqual(
            StaticPluginLayout.resolvedHeight(
                preferredHeight: 100,
                baseHeight: 200,
                visibleScreenHeight: 900,
                fallbackMaximumHeight: 332
            ),
            200
        )
    }

    func testClampsToSeventyPercentOfVisibleScreen() {
        XCTAssertEqual(
            StaticPluginLayout.resolvedHeight(
                preferredHeight: 800,
                baseHeight: 200,
                visibleScreenHeight: 600,
                fallbackMaximumHeight: 332
            ),
            420
        )
    }

    func testUsesExistingFallbackWithoutScreenHeight() {
        XCTAssertEqual(
            StaticPluginLayout.resolvedHeight(
                preferredHeight: 460,
                baseHeight: 200,
                visibleScreenHeight: nil,
                fallbackMaximumHeight: 332
            ),
            332
        )
    }
}
```

- [ ] **Step 3: Verify RED in the focused Swift package harness**

Run:

```bash
DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer xcrun swift test --package-path /tmp/atoll-plugin-tests.9qo3gG
```

Expected: compilation fails with `cannot find 'StaticPluginLayout' in scope`.

### Task 4: Implement adaptive static-plugin height

**Files:**
- Modify: `DynamicIsland/models/StaticPlugins/StaticPluginManifest.swift`
- Modify: `DynamicIsland/ContentView.swift`

- [ ] **Step 1: Add the pure height resolver**

Append to `StaticPluginManifest.swift`:

```swift
enum StaticPluginLayout {
    static let maximumVisibleScreenFraction: CGFloat = 0.7

    /// 将插件请求高度限制在默认刘海高度和当前屏幕安全高度之间。
    static func resolvedHeight(
        preferredHeight: CGFloat,
        baseHeight: CGFloat,
        visibleScreenHeight: CGFloat?,
        fallbackMaximumHeight: CGFloat
    ) -> CGFloat {
        let maximumHeight = visibleScreenHeight.map {
            max(baseHeight, $0 * maximumVisibleScreenFraction)
        } ?? max(baseHeight, fallbackMaximumHeight)
        return min(max(preferredHeight, baseHeight), maximumHeight)
    }
}
```

- [ ] **Step 2: Call the resolver from ContentView**

Replace the body of `staticPluginPreferredHeight(baseSize:)` after its guard with:

```swift
return StaticPluginLayout.resolvedHeight(
    preferredHeight: CGFloat(preferredHeight),
    baseHeight: baseSize.height,
    visibleScreenHeight: NSScreen.main?.visibleFrame.height,
    fallbackMaximumHeight: baseSize.height + statsAdditionalRowHeight
)
```

- [ ] **Step 3: Verify GREEN and regressions**

Run:

```bash
DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer xcrun swift test --package-path /tmp/atoll-plugin-tests.9qo3gG
python3 -m unittest tests.test_privacy_configuration tests.test_timer_lifecycle
git diff --check
```

Expected: 25 focused Swift tests and seven Python regression tests pass; the diff check is clean.

- [ ] **Step 4: Commit and push the host fix**

```bash
git add DynamicIsland/ContentView.swift DynamicIsland/models/StaticPlugins/StaticPluginManifest.swift DynamicIslandTests/StaticPluginPackageValidatorTests.swift
git commit -m "fix(plugins): respect requested tab height"
git push origin feat/static-web-plugins
```

Expected: upstream PR #781 updates without adding personal workflow or process documents.

### Task 5: Validate CI and rebuild the personal app

**Files:**
- No new source files.

- [ ] **Step 1: Reopen the fork validation PR and wait for checks**

```bash
gh pr reopen 1 --repo Sq-List/Atoll
gh pr checks 1 --repo Sq-List/Atoll --watch --interval 10
```

Expected: changelog validation and macOS 15/26 builds pass. Close fork PR #1 after recording the run URL.

- [ ] **Step 2: Apply the host commit to the personal build branch**

```bash
git checkout codex/static-plugin-app-build
git cherry-pick "$(git rev-parse feat/static-web-plugins)"
git push origin codex/static-plugin-app-build
```

Expected: `Personal Static Plugin App` runs for the new head and uploads a replacement ZIP.

- [ ] **Step 3: Confirm isolation**

```bash
gh pr view 781 --repo Ebullioscopic/Atoll --json headRefOid,isDraft,state,url
git status --short --branch
```

Expected: PR #781 contains the host fix but not the personal workflow/design documents; the personal branch retains its workflow; the unrelated untracked documents remain untouched.

- [ ] **Step 4: Hand off replacement steps**

Tell the user to download the new personal app artifact, launch it without overwriting `/Applications`, then import `/Users/sqlist/Project/other/AtollDeveloperTools/DeveloperTools.atollplugin` and confirm Replace from version 1.0.0 to 1.0.1.
