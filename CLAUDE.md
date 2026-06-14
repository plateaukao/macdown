# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

MacDown is an open-source Markdown editor for macOS, written in Objective-C / Cocoa. It is a document-based AppKit application (not a modern Swift/SwiftUI app) and uses the legacy `WebView` (WebKit) for preview, not `WKWebView`.

## Setup

After cloning, from the repo root:

```
git submodule update --init                  # pulls Dependency/prism
bundle install                               # installs the pinned CocoaPods via Bundler
bundle exec pod install                      # installs Pods, generates MacDown.xcworkspace
make -C Dependency/peg-markdown-highlight    # builds the C editor-highlighter library
```

Always open and build `MacDown.xcworkspace` (the workspace), **never** `MacDown.xcodeproj` directly — the project alone won't link against Pods. CocoaPods must be run through Bundler; a system CocoaPods older than `Gemfile.lock` is unsupported.

If builds break after a pull, re-run `git submodule update` and `bundle exec pod install`.

## Build & test

```
# Build + run the full test suite (CI does exactly this)
xcodebuild -workspace MacDown.xcworkspace -scheme MacDown test | xcpretty

# Run a single test class or method
xcodebuild -workspace MacDown.xcworkspace -scheme MacDown test \
  -only-testing:MacDownTests/MPUtilityTests
xcodebuild -workspace MacDown.xcworkspace -scheme MacDown test \
  -only-testing:MacDownTests/MPUtilityTests/testSomeMethod
```

`MacDown` is the only shared scheme; building it also builds the `macdown-cmd` and `MacDownTests` targets. Tests are XCTest (`MacDownTests/MP*Tests.m`).

The app version is **not** hard-coded: `Tools/generate_version_header.sh` reads `Tools/version.txt` plus git state and writes `version.h` (consumed via `MPGlobals.h`) as a build phase.

## Targets

- **MacDown** — the editor app.
- **macdown-cmd** — the `macdown` CLI (installed to `/usr/local/bin/macdown`). It does **not** render anything itself; it serializes the files/piped content to open into the shared `NSUserDefaults` suite (`com.uranusjr.macdown`) and signals the running app. See `macdown-cmd/` and `NSUserDefaults+Suite`.
- **MacDownTests** — unit tests.

## Architecture

All classes use the `MP` prefix. Source lives under `MacDown/Code/`, grouped by role (`Application`, `Document`, `Preferences`, `View`, `Utility`, `Extension`).

### The rendering pipeline (the heart of the app)

`MPDocument` is the per-window controller (one `NSDocument` per open file). It owns three collaborators and wires them together:

- **`MPEditorView`** (an `NSTextView` subclass) — the text editor.
- **`HGMarkdownHighlighter`** — *editor-side* syntax highlighting of the Markdown source, driven by the C library built from `Dependency/peg-markdown-highlight`. This is separate from preview highlighting.
- **`MPRenderer`** — *preview-side* Markdown→HTML rendering.
- A legacy **`WebView`** — displays the rendered HTML preview.

`MPRenderer` converts Markdown to HTML using **Hoedown** (a C library, pulled in via CocoaPods and customized through `Code/Extension/hoedown_html_patch.{c,h}`), then wraps the HTML fragment in a full document using the **Handlebars** template `MacDown/Resources/Templates/Default.handlebars` (rendered via `handlebars-objc`). Style/script `<link>`/`<script>` tags are modeled as `MPAsset` objects and injected into the template's `styleTags`/`scriptTags`.

`MPDocument` is both the renderer's `dataSource` (supplies the Markdown text and HTML title) and its `delegate` (supplies every rendering option — Hoedown extension flags, SmartyPants, TOC, preview style name, front-matter detection, syntax highlighting, Mermaid, Graphviz, MathJax, code-block accessory type). When rendering finishes, the renderer calls back `renderer:didProduceHTMLOutput:`, and `MPDocument` loads the result into the `WebView`. Read `MPRenderer.h` for the full `MPRendererDataSource` / `MPRendererDelegate` contract before changing rendering behavior.

Note the MathJax workaround: `MathJax.js` is vendored and a `WebResourceLoadDelegate` is wired into `MPDocument` to work around an upstream bug — see the comment block at the top of `MPRenderer.m` before touching either.

### Preferences

`MPPreferences` subclasses **`PAPreferences`**: properties are declared `@dynamic`-style and automatically backed by `NSUserDefaults`, accessed through a singleton (`[MPPreferences sharedInstance]`). To add a setting, add a property to `MPPreferences.h` — no manual getter/setter or key constant needed. The settings window uses **MASPreferences** with one `MP*PreferencesViewController` per pane (General, Editor, Markdown, HTML/Rendering, Terminal).

### App lifecycle & other pieces

- `MPMainController` is the `NSApplicationDelegate`.
- `MPPlugInController` / `MPPlugIn` implement a plugin system (`.dukapp`-style bundles).
- `Code/Extension/` holds Foundation/AppKit category extensions (e.g. `NSString+Lookup`, `NSColor+HTML`, `NSObject+HTMLTabularize`) — these are the most heavily unit-tested parts.
- `Code/Utility/MPUtilities.{h,m}` centralizes filesystem paths for user-installed styles/themes/plugins.

### Bundled web/render resources (`MacDown/Resources/`)

- `Styles/*.css` — **preview** stylesheets shown in the WebView.
- `Themes/*.style` — **editor** color themes applied by `HGMarkdownHighlighter` (a different concept from preview Styles).
- `Prism/` — syntax highlighting for preview code blocks, sourced from the `Dependency/prism` submodule.
- `Extensions/` — injected JS/CSS for Mermaid (`mermaid.min.js`), Graphviz (`viz.js`), task lists, etc.
- `MathJax/`, `Templates/Default.handlebars`.

## Conventions (enforced; see CONTRIBUTING.md)

- **80-column limit** on all lines (exception: long URLs in comments).
- **Allman brace style** — braces on their own line.
- **4 spaces**, never tabs; no trailing whitespace; trailing newline at EOF.
- Omit braces for single-statement blocks, *unless* part of an `if`/`else if`/`else` chain (then all branches match).
- Prefer implicit boolean tests: `if (str.length)` not `if (str.length != 0)`. But use explicit `== 0`/`!= 0` when comparing a genuine numeric zero (`NSRange` location, coordinates, etc.).
- In multi-line conditionals, put logical operators at the **start** of the continuation line.
- Rebase feature branches onto `master` before opening a PR; `.xib` and project files merge badly, so prefer small commits.
