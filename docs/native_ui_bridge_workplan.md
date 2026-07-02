# Native UI Bridge — Architecture & Workplan

Goal: replace the native Views-based browser chrome (tab strip, toolbar,
sidebar) with a web-rendered UI (HTML/CSS/JS — think Zen Browser's vertical
tabs and visual design) running on top of unmodified Blink/V8, so the browser
engine stays 100% stock Chromium/ungoogled-chromium while the UI is as
flexible as any modern web app.

## Major update (2026-07-03): Chromium 149 already has this — codename "Webium"

While implementing Phase 1 we discovered `chrome/browser/ui/webui_browser/`
and `chrome/browser/resources/webui_browser/` — an internal-Google prototype
(feature flag `features::kWebium`, `chrome://flags/#webium`, "Webium
Prototype Browser", gated to Canary/dev builds) that is **exactly** the
architecture this document independently arrived at: a `BrowserWindow`
subclass (`WebUIBrowserWindow`) that hosts the entire browser chrome as a
Lit-based WebUI frontend, talking to the browser process over `browser.mojom`.
It is materially more complete than anything we'd planned for Phase 1:

- **Full chrome, not a stub**: frameless custom-drawn window (its own
  minimize/maximize/restore/close buttons, drawn in the page), tab strip with
  drag-to-reorder, back/forward with long-press history menu, omnibox
  (`cr-searchbox`), security icon, app menu, avatar/profile menu, side panel,
  bookmark bar, extensions bar/toolbar, fullscreen (tab- and browser-level).
  ~30 C++ files under `chrome/browser/ui/webui_browser/` +
  ~25 TS/Lit files under `chrome/browser/resources/webui_browser/`
  (`app.ts` is the root element; `tab_strip/`, `webview.ts`, `side_panel.ts`,
  `bookmark_bar.ts`, `extensions_bar.ts` are the major sub-components).
- **Answers Phase 0's open compositing question**: real tab content is
  attached into the trusted WebUI page via a privileged internal API —
  `WebviewElement` (`webview.ts`) creates an `<iframe>` and calls
  `chrome.browser.attachIframeGuest(guestId, iframeContentWindow)` to attach
  the tab's actual guest `WebContents` into it (there's also a newer, more
  efficient `enableSurfaceEmbed` mode that skips the iframe indirection via
  direct compositor-surface embedding — check `loadTimeData`-gated code paths
  in `webview.ts` for both). This is the real answer to "can a WebView-hosted
  chrome coexist with real tab compositing" — yes, and here's the mechanism.
- **Mojo surface** (`chrome/browser/ui/webui_browser/browser.mojom`):
  `PageHandlerFactory`, `Page` (browser→renderer push), `PageHandler`
  (renderer→browser calls: window controls, app/profile menus, tab strip
  inset, back/forward menu, fullscreen), `GuestHandler` (the guest-attachment
  plumbing `webview.ts` calls into). `WebUIBrowserWindow` itself
  (`webui_browser_window.h`) is a `BrowserWindow` subclass — confirms
  `BrowserWindow::CreateBrowserWindow()` (`browser_window_factory.cc`) is
  exactly the injection point our Phase 1 was aiming for; Webium already
  occupies it, gated behind `webui_browser::IsWebUIBrowserEnabled()`.
- **Not finished**: many methods are `NOTIMPLEMENTED()` stubs — confirmed by
  running it (`--enable-features=Webium,AttachUnownedInnerWebContents,ExtensionsMenuAccessControl`):
  window/tab creation and navigation work, but `WebUIStubLocationBar` (no real
  `OmniboxController`), status bubbles, content-settings icons, workspace
  queries, and several toolbar-icon update paths are unimplemented no-ops.

**Decision (confirmed with user 2026-07-03): build on Webium rather than
write a parallel `BrowserWindow` subclass.** It's a working, non-trivial
head start on precisely this architecture from people with full Chromium
context; duplicating it would be wasted effort. Tradeoff accepted: this ties
us to an internal/unstable Google prototype (not a public API, could be
renamed, restructured, or deleted in any future Chromium roll) rather than a
from-scratch implementation we fully control — worth re-evaluating if a
future Chromium version removes or significantly changes it.

Revised Phase 1: study Webium's remaining gaps (location bar/omnibox
integration is the biggest — `WebUIStubLocationBar` has no real
`OmniboxController`), fill them in, then build Zen-style vertical tabs as a
`tab_strip/` layout variant on top of the existing `TabStripElement`/
`browser.mojom` plumbing rather than inventing new plumbing. The sections
below (Chromium's WebUI+Mojo mechanism, patch-layering discipline) remain
accurate background — Webium itself is built on exactly that mechanism.

## Why this design, not Vivaldi's or Brave's

Researched both. Neither is a direct template as-is:

- **Vivaldi** hosts its entire UI (`window.html`) inside a repurposed
  `extensions::AppWindow` (the Chrome Packaged Apps platform), and exposes
  native functionality via a family of private extension APIs
  (`vivaldi.tabsPrivate`, `vivaldi.windowPrivate`, `vivaldi.bookmarksPrivate`,
  etc.) routed through `ExtensionFunctionDispatcher`. This proves the general
  shape works (web-rendered chrome around real tab `WebContents`), but the
  Chrome Apps platform it's built on is deprecated and being actively removed
  upstream — building fresh on it means fighting upstream deletion, not just
  patch churn. Not viable as a foundation.
- **Brave** does the opposite for its chrome: vertical tabs and the sidebar
  are 100% native Views C++ (`BraveBrowserView : public BrowserView`,
  `BraveVerticalTabStripRegionView`, etc.) — exactly what we're trying to
  avoid rewriting/maintaining. Not useful as a UI template, but two things
  from Brave *are* directly reusable:
  - Their **patch layering discipline** (`brave-core/docs/patching_and_chromium_src.md`):
    prefer pure-addition files → subclassing with minimal upstream `virtual`/
    `friend` patches → `chromium_src/` path-shadowing for full-file swaps →
    raw diff patches only as a last resort. This maps directly onto how
    ungoogled-chromium's own patch set is structured, so we can adopt it for
    the bridge code without inventing new tooling.
  - Their non-native surfaces (New Tab Page, Wallet, Rewards, Reader Mode)
    use **vanilla Chromium WebUI + Mojo** — proving that mechanism is the
    supported, low-friction way to expose typed C++ APIs to a privileged web
    page in a Chromium derivative.

**Decision: build the entire browser chrome as a `chrome://`-scheme WebUI
page, bridged to the browser process via typed Mojo interfaces**, using the
exact mechanism Chromium's own `chrome://tab-search` / `chrome://history` /
`chrome://bookmarks` pages use. This is fully open, actively maintained by
upstream, and survives rebases better than anything extension-API- or
AppWindow-based.

## High-level architecture

```
BrowserView (or a new BrowserWindow impl)
 └─ views::WebView  →  navigates to  chrome://browser-ui/
                          │
                          ├─ WebUIController (MojoWebUIController subclass)
                          │    binds PageHandlerFactory
                          │
                          ├─ per-subsystem PageHandler impls (C++, browser process)
                          │    - TabsPageHandler      → TabStripModel
                          │    - WindowPageHandler     → Browser, BrowserWindow
                          │    - BookmarksPageHandler  → BookmarkModel
                          │    - HistoryPageHandler    → HistoryService
                          │    - DownloadsPageHandler  → DownloadManager
                          │    - SettingsPageHandler   → PrefService
                          │    - OmniboxPageHandler     → AutocompleteController
                          │    - ExtensionsPageHandler  → extensions::ExtensionRegistry
                          │
                          └─ Page (browser → renderer push channel, per handler)

  Tab content itself stays as ordinary content::WebContents, embedded via
  views::WebView instances the UI page controls indirectly (create/destroy/
  reparent driven through TabsPageHandler, actual pixels composited by
  Chromium as normal — no change to how tab content renders).
```

Two open design questions to resolve early (see Phase 0):

1. **Where does the outer window frame come from?** Two options:
   - (a) Keep `BrowserView`/native `views::Widget` for the OS-level window
     (titlebar/resize handles, like Vivaldi's optional native-titlebar mode),
     and swap out its children below the frame for a single `views::WebView`.
     Lower risk, faster to bootstrap.
   - (b) Go frameless and let the web UI draw a custom titlebar too (closer
     to Zen's look). Requires wiring OS drag-region / min/max/close through
     the bridge (`WindowPageHandler.StartWindowDrag`, etc.) — Chromium
     already has support for this via `-webkit-app-region: drag` in WebUI/PWA
     window contexts, worth reusing rather than inventing.
   - Recommendation: start with (a), add draggable-region support to reach
     (b) once the bridge is stable.
2. **Trusted vs. untrusted split.** The main chrome page should be
   `chrome://` (trusted, full Mojo bindings). If we ever embed
   extension-provided panels or anything web-influenced inside the chrome
   itself, that piece should be `chrome-untrusted://` per Chromium's
   documented pattern, not the main page.

## Phase 0 — Spike (prove the shell works)

- [x] Register a minimal `chrome://browser-ui` `WebUIConfig` +
      `MojoWebUIController` that serves a static "hello world" page.
- [ ] New `BrowserWindow` subclass (start as a thin subclass of `BrowserView`,
      per option (a) above) that replaces the native tab strip/toolbar region
      with a single `views::WebView` navigated to `chrome://browser-ui`.
- [ ] Confirm a real tab's `WebContents` can still be shown/resized correctly
      underneath/beside the WebUI-hosted chrome (this is the trickiest
      compositing question — validate early, it's the one thing that can
      invalidate the whole approach).
- [x] One trivial Mojo round-trip: a button in the WebUI page that calls
      `PageHandler.CreateTab(url)` and actually opens a tab.

### Status (2026-07-03): Mojo round-trip validated, window-replacement not yet started

Landed at `chrome/browser/ui/webui/browser_ui/` (+ resources at
`chrome/browser/resources/browser_ui/`). `chrome://browser-ui` loads as a
normal WebUI tab; its "Open New Tab" button calls
`browser_ui.mojom.PageHandler.CreateTab()`, which resolves the hosting
window via `webui::GetBrowserWindowInterface()` (cycle-safe — does **not**
depend on `//chrome/browser/ui`) and calls
`BrowserWindowInterface::OpenURL()` (`content::PageNavigator`). Verified
end-to-end with a real build: WebDriver click → tab count goes 1→2.
Confirms the core WebUI+Mojo bridge mechanism from the architecture decision
works as expected. What's *not* done yet: this page doesn't replace any
native chrome — Phases "new `BrowserWindow` subclass" and "WebView hosting
real tab compositing" are still open, and are the next real risk to retire.

Three non-obvious wiring steps this took, worth remembering for Phase 1
(each is a separate registration point, and skipping any one fails
differently):
1. **Resource ID range**: `tools/gritsettings/resource_ids.spec` needs a
   manually-picked, non-colliding `"includes"` id/size entry for every new
   `.grd` — grit hard-fails on first build without it (find a same-sized gap
   between alphabetically-adjacent existing entries).
2. **Resource repacking is NOT auto-discovered.** Depending on the feature's
   `:resources` target from `chrome/browser/resources/BUILD.gn`'s
   `public_deps` is necessary (for build ordering) but **not sufficient** —
   the actual runtime `resources.pak` is assembled by a literal, hand-maintained
   `sources` list in `chrome/chrome_paks.gni` (comment: "New paks should be
   added here by default"). Miss this and the page 404s with `ERR_FAILED`
   even though everything else (including the `WebUIController` itself)
   works — the failure is silent and looks identical to a registration bug,
   not a resources bug. Diagnosed by round-tripping `LOG(ERROR)` statements
   through the constructor and manually inspecting `.pak` contents with
   `tools/grit/pak_util.py list-id`.
3. **`AddWebUIConfig` and `RegisterWebUIControllerInterfaceBinder` are two
   separate registrations.** The former makes the URL resolve to your
   `WebUIController`; without the latter the page loads fine but any Mojo
   call from JS kills the renderer with "No binder found for interface ...".
   Reset_password-style single-interface features register the latter in
   `chrome/browser/chrome_browser_interface_binders_webui_parts_desktop.cc`
   (or `..._features.cc` if the feature is buildflag-gated).

Also found and fixed (see `patches/core/ungoogled-chromium/disable-google-host-detection.patch`,
reported upstream-worthy): the patch deletes `variations::kClientDataHeader`'s
definition but missed two `net::URLRequest*` overloads that reference it,
added by Chromium after the patch was last synced to this version — link
error, not a spike-specific issue, would break any build of this fork at
149.0.7827.200 regardless of these changes.

Exit criterion: a window that looks empty/ugly but has a web-rendered chrome
controlling one real Chromium tab. Everything after this is breadth, not new
risk.

## Phase 1 — Core subsystem bridges (mojom interfaces)

Mirror Vivaldi's private-API taxonomy (it's a validated split of concerns),
but as typed `.mojom` interfaces instead of extension functions. Suggested
location: `chrome/browser/ui/webui/browser_ui/*.mojom`, one file per
subsystem, all bound from the single `BrowserUIController`.

| Interface | Backs onto | Core methods (illustrative) |
|---|---|---|
| `TabsPageHandler` / `TabsPage` | `TabStripModel` | CreateTab, CloseTab, ActivateTab, MoveTab, DuplicateTab, MuteTab, GetTabs; push: OnTabCreated/Removed/Moved/Updated |
| `WindowPageHandler` / `WindowPage` | `Browser`, `BrowserWindow` | NewWindow, CloseWindow, Minimize/Maximize/Restore, StartWindowDrag, SetFullscreen, GetWindowState; push: OnWindowStateChanged |
| `BookmarksPageHandler` | `BookmarkModel` | Get/Create/Update/Remove/Move bookmark & folder; push: OnBookmarkChanged |
| `HistoryPageHandler` | `HistoryService` | QueryHistory, DeleteUrls; push: OnHistoryChanged |
| `DownloadsPageHandler` | `DownloadManager` | ListDownloads, Pause/Resume/Cancel, OpenDownload; push: OnDownloadUpdated |
| `SettingsPageHandler` | `PrefService` | GetPref, SetPref, ObservePrefs; push: OnPrefChanged |
| `OmniboxPageHandler` | `AutocompleteController` | QuerySuggestions, OpenMatch; push: OnSuggestionsUpdated |
| `ExtensionsPageHandler` | `extensions::ExtensionRegistry` | ListExtensions, TogglePinned, OpenPopup |
| `ThemePageHandler` | our own theming layer | GetTheme, SetAccent, ObserveThemeChanges (needed since we're replacing Chrome's native theme entirely) |

Each handler: a small C++ class in `chrome/browser/ui/webui/browser_ui/`,
constructed per-WebUI-instance with a `Browser*`/`Profile*`, implementing the
generated mojom interface, doing the real work by calling into existing
Chromium services (no new business logic — this layer is purely a typed
translation, matching Brave's NTP handler pattern).

Work per interface: define `.mojom` → implement handler → register binder in
the browser-ui equivalent of `chrome_browser_interface_binders_webui.cc` →
generated `.mojom-webui.ts` consumed by the frontend.

## Phase 2 — Frontend chrome application

- Build the actual UI (vertical tabs, sidebar, toolbar) as a normal
  TypeScript/whatever-framework app under `chrome/browser/resources/browser_ui/`,
  bundled via the existing grit/grd resource pipeline (`WebUIDataSource`,
  `AddResourcePath`, CSP via `webui::SetupWebUIDataSource()`).
- Not a Zen/Firefox code port (Zen's UI is MPL-licensed XUL/CSS, not directly
  reusable in a BSD Chromium tree) — reimplement the visual design from
  scratch against the Mojo-backed data model from Phase 1.
- State management: `Page` push interfaces (Phase 1) feed a reactive
  store; avoid polling.

## Phase 3 — Fork-maintenance tooling

- Adopt Brave's layering priority for every native-side change:
  1. new files under our own namespace (bridge handlers, mojom) — zero patch
     cost, never conflicts.
  2. subclass + minimal upstream `virtual`/`friend` patches (e.g. whatever's
     needed on `BrowserView`/`Browser` to let our `BrowserWindow` subclass
     hook in).
  3. `chromium_src/`-style path-shadowed full-file overrides for anything
     that needs a substantive behavior swap (e.g. suppressing native
     tab-strip construction).
  4. raw patch files (already how this repo's `patches/` works) only for
     genuinely small, surgical hook points.
- Track upstream deprecation/removal of anything we depend on (Mojo/WebUI
  itself is core infra, low risk; avoid Chrome-Apps/extension-only APIs
  entirely, per the Vivaldi finding).

## Open risks to validate before committing further

1. **Compositing tab `WebContents` next to/around a WebUI-hosted chrome** —
   the one genuinely novel piece of layout work; do this first (Phase 0).
2. **Rebase cost** of the `BrowserWindow` subclass and any required upstream
   `virtual`/`friend` patches as Chromium versions roll — should be small if
   we hold the line on layering discipline (Phase 3), but needs a couple of
   version bumps to confirm in practice.
3. **Frontend framework choice** for the chrome app — not decided here;
   pick based on bundle size/CSP constraints inside a WebUI context (no
   `eval`, strict CSP by default), not general web-app preferences.

## Sources

Findings underlying this plan came from public source inspection (Vivaldi
C++ mirrors, brave-core), Brave's own architecture docs
(`brave-core/docs/patching_and_chromium_src.md`), and Chromium's WebUI
documentation (`docs/webui/webui_explainer.md`,
`docs/webui/webui_in_chrome.md`). Vivaldi's frontend JS itself is
closed-source and was not and cannot be inspected; only its native-side
integration code is public.
