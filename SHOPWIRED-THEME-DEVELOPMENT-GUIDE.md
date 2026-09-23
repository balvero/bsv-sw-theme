# ShopWired Theme Development — Agent Guide

A general-purpose reference for an AI coding agent working on **any** ShopWired Twig
theme — not specific to this theme or to any single feature. Read this before making
changes to a ShopWired theme you haven't worked in before; it front-loads the platform
quirks and CLI gotchas that otherwise cost real debugging time.

For a worked example of porting one specific feature (Algolia search) between themes,
see `ALGOLIA-SEARCH-IMPLEMENTATION-GUIDE.md` alongside this file — that document assumes
everything here.

## What a ShopWired theme is

A ShopWired theme is a Twig template set plus assets, hosted and rendered server-side by
ShopWired. There is no local dev server that renders the theme — you edit files locally
(or in the browser-based Code Editor) and either sync them up to the platform to preview
against the merchant's real store data, or edit directly in the admin.

## Two ways to edit theme files — pick the right one

### 1. Theme Engine CLI (`shopwired-theme`) — preferred for AI-agent work

An npm package (`shopwired-theme`, invoked as the `shopwired-theme` command once
installed globally) that syncs a local folder with a theme on ShopWired's servers.
This is what you want when working from a terminal/coding agent, since it lets you
`Read`/`Edit`/`grep` real files, run local validators (`node --check`, `sass`), and use
git for history — none of which the browser Code Editor gives you.

Setup: `shopwired-theme config` (or pass `-k <apiKey> -s <apiSecret> -t <themeId>` flags)
writes a `config.json` into the current directory containing `apiKey`, `apiSecret`, and
`themeId`. **This file must never be committed to git** — add it to `.gitignore`
immediately in any new theme project. It's a live credential file, not theme content.

Commands:

| Command | Effect |
|---|---|
| `shopwired-theme download [files...]` | Pull files from the theme down to local disk. No arguments = download everything. |
| `shopwired-theme upload <files...>` | Push local files up to the theme. Required arguments — no "upload everything" shortcut. |
| `shopwired-theme remove <files...>` | Delete files from the theme (e.g. after deleting them locally). |
| `shopwired-theme watch` | Watch the local folder and auto-upload on save. Useful for a human iterating in an editor; less useful for an agent doing discrete edits. |

Every argument to `upload`/`download`/`remove` is a **literal relative file path** — there
is no glob/wildcard expansion and no directory argument that means "everything in this
folder." List every file explicitly.

### 2. ShopWired admin's built-in Code Editor

**Website → Themes → find the theme → "code editor"** (for the active theme) or
**"actions → code editor"** (for an unpublished/inactive theme). A left-hand file tree
lets you browse and edit files directly in the browser; `Ctrl/Cmd+S` saves. New files can
be created here too, restricted to `.twig`, `.js`, `.css`, `.scss`, `.json` extensions.

Use this path when: you (or the user) are working interactively in a browser and want to
hand an AI agent copy-paste-ready snippets rather than CLI access — see the Algolia guide
for an example of writing a guide in that style. As an agent with terminal/file access,
default to the CLI instead.

**Before any significant edit session on a theme that's already live on a real store**,
ShopWired's own docs recommend duplicating the theme first as a backup
(Website → Themes → duplicate, or "how to take a backup of your theme" in their help
docs). Mention this to the user if you're about to make a large structural change to a
theme that's actively serving traffic.

## Critical gotchas — read before deploying anything

These are not hypothetical; each one caused a real problem in a prior session and cost
time to diagnose.

1. **An "Uploaded X" success message does not prove the file reached the theme you
   intended.** If `config.json` credentials were just changed, inherited from someone
   else, or you have any doubt which theme they point at, verify with a round-trip before
   trusting anything else you do:
   ```
   cp <file> /tmp/verify-copy
   shopwired-theme upload <file>
   shopwired-theme download <file>
   diff /tmp/verify-copy <file> && echo VERIFIED
   ```
   If this doesn't come back clean, stop everything else and get the credentials/theme ID
   confirmed with the user before proceeding — every prior upload in the session may have
   gone to the wrong theme.
2. **The CLI occasionally fails a single file with `ECONNRESET` / "socket hang up."**
   This is transient network flakiness in the CLI's HTTP layer, not a real error — just
   retry the same `upload`/`download` command for the file(s) that failed. Don't treat it
   as a sign something is broken.
3. **A local file deletion is not reflected remotely by `upload`** (there's nothing to
   upload). Use `shopwired-theme remove <path>` explicitly to delete a file from the live
   theme.
4. **SCSS compiles as a single unit.** All SCSS partials referenced from `theme.scss` are
   compiled together into one `theme.css`. A syntax error **anywhere** in that chain
   blocks the *entire* CSS file from regenerating — not just the broken partial. This
   means a small mistake in a new partial you're adding can silently stop unrelated,
   already-working styles from updating on the live site. Always validate a partial in
   isolation before uploading:
   ```
   npx --yes sass --no-source-map assets/scss/_your-partial.scss /tmp/out.css
   ```
   This only catches pure-SCSS syntax errors — it won't catch ShopWired-specific
   functions like `setting-value()` that only exist in the platform's compiler, so a file
   using those will fail this local check even when it's fine live. Read the file's
   existing style (does it already use platform functions?) before assuming a local
   compile failure means the file is broken.
   If you need to debug an error that only shows up on the live compiler, ShopWired has
   an Advanced theme setting, "Show SCSS compilation errors in the browser console" — ask
   the user to enable it if you're stuck, then re-save `theme.scss` to force a
   recompile and check the browser console. Reported line numbers can be slightly off and
   errors near a partial boundary sometimes get attributed to the wrong file — check
   neighboring partials in the import order too.
5. **Never edit `theme.json` casually.** It declares the sections/blocks/color schemes
   available in the theme editor. ShopWired's own docs are explicit: *"Changing the
   theme.json file after installation of a theme will reset any sections already
   configured on your theme."* If a merchant has already customized their section layout
   through the theme editor, editing this file can wipe that configuration. Only touch it
   when the user explicitly asks you to add/change a section or block definition, and
   confirm with them first if the theme is already live/configured.
6. Validate JS syntax before uploading, same reasoning as SCSS:
   ```
   node --check assets/js/application.js
   ```

## Theme file structure

```
theme-root/
├── settings.json        # theme-wide settings schema (see below) — safe to edit freely
├── theme.json            # sections/blocks/color schemes — DO NOT edit casually (see above)
├── config.json            # shopwired-theme CLI credentials — gitignored, never commit
├── assets/
│   ├── fonts/             # icon fonts (sw-icons etc.) and any custom webfonts
│   ├── images/             # static images, referenced via the `asset_url()` Twig function
│   ├── js/
│   │   └── application.js  # the theme's own JS — inits, carousels, basket, modals, etc.
│   │                         # (plus whatever custom scripts you're adding)
│   └── scss/
│       ├── theme.scss       # entry point — @imports every other partial, compiles to theme.css
│       └── _*.scss           # partials (leading underscore = not compiled standalone)
└── views/
    ├── home.twig            # homepage template
    ├── search.twig, product.twig, category.twig, ...   # one file per route/page type
    ├── macros/
    │   ├── html.twig         # generic reusable HTML-producing macros
    │   └── theme.twig         # theme-specific macros (icons, menus, price formatting, etc.)
    ├── partials/              # smaller reusable snippets, pulled in via {% include %}
    │   ├── header.twig, footer.twig, item.twig, items.twig, product_filters.twig, ...
    └── templates/
        ├── master.twig        # the actual <html>/<head>/<body> shell every page extends into
        ├── page.twig, collection.twig, ...   # shared layout templates other views extend
```

`views/sections/` exists on newer "Version 5" themes that use theme.json's
section/block system for merchant-configurable page building; older themes (like the one
this guide's sibling Algolia doc was extracted from) may not have it and instead compose
pages more directly through `templates/`/`partials/`.

## Twig on ShopWired — what's different from vanilla Twig

- **`global`** is the main data object exposed to every template: `global.theme.settings`
  (almost universally aliased to a local variable `gts` — grep for
  `{% set gts = global.theme.settings %}`), `global.customer`, `global.basket`,
  `global.categories`, `global.business`, `global.current_url`/`current_path`, etc.
  Check "ShopWired objects" in their docs (or just grep the theme for `global.`) for the
  full surface — it's large and theme-dependent on what's populated per page.
- **`asset_url()`** resolves a path under `assets/` to its live CDN URL — always use it
  for referencing the theme's own images/compiled CSS, never a hardcoded relative path.
- **Macros**: imported per-file with `{% import 'macros/html.twig' as html %}` and
  `{% import 'macros/theme.twig' as theme %}` near the top of most templates. Common ones
  you'll use constantly: `theme.sw_icon('icon-name')` (renders an `<i class="sw-icon-...">`
  from the theme's icon font — check `assets/scss/_fonts.scss` for which icon names
  actually exist before using one from a *different* theme, they're not guaranteed to
  match), `html.stylesheet(url)`, `html.script(url)`, `html.image(...)`,
  `html.input(...)`.
- **Blocks and `{% extends %}`** work like standard Twig, with one execution-order detail
  worth knowing explicitly (not officially documented by ShopWired, verified empirically):
  when template A extends B extends C, **each template's own top-level code (outside any
  block) runs leaf-first**, before the base template (`master.twig`, typically) produces
  any output. Concretely: if `master.twig` sets `{% set gts = global.theme.settings %}`
  at its own top level, and a *child* template several `extends` deep also has top-level
  code that reads `gts`, that child's `gts` read happens **before** `master.twig`'s own
  `{% set %}` has run — `gts` will be `null`/undefined at that point unless the child
  template redeclares it itself (`{% set gts = global.theme.settings %}` — safe to do
  redundantly, `global` itself is a true global regardless of render order). This has
  caused a real, silent bug before (a feature flag computed from `gts.xxx` at a child
  template's top level always evaluated false) — if you add any top-level `{% set %}` in
  a child template that reads `gts`, redeclare `gts` locally first, don't assume the
  parent already set it up by the time your code runs.
- **`{% if not algoliaSearch %}`-style guards work fine even when the variable is only set
  in *some* templates in the extends chain** (e.g. only in `search.twig`, not in
  `category.twig` which extends the same shared `collection.twig`) — Twig treats an
  undefined variable as falsy rather than erroring, so this is a safe, idiomatic way to
  add page-type-specific behavior to a shared template.

## settings.json — theme-wide configuration schema

Structure: `{ "sections": [ { title, description?, help?, icon?, resettable?, group?,
settings: [ ... ] }, ... ] }`. Max 75 sections, 250 settings per section.

Common setting `type` values: `text`, `textarea`, `rich_text`, `checkbox`, `radio`,
`drop_down`, `color`, `slider` (`minValue`/`maxValue`/`step`/`valueSuffix`), `image`,
`gallery`, `logo`, `favicon`, `google_font`, `category`/`product`/`page`/`blog_post`
(entity pickers), `link_list`, `payment_method`, `code` (a Twig snippet editor), plus
non-interactive `header`/`paragraph` for in-editor documentation. `livePreview: true` (or
`cssCustomProperty`) makes the editor preview update without a full page reload.
`parentName`/`parentValue` implement conditional show/hide of one setting based on
another's value, within the same section.

Access in Twig: `{{ global.theme.settings.your_setting_name }}` (via `gts`, per above).
Access in SCSS: settings with `scssVariable: true` become compiled Sass variables — check
an existing theme's `_variables.scss`/`_shopwired.scss` for the naming convention in use
(commonly `$color_<setting_name>` for color settings) rather than guessing.

Adding a new settings **section** is always safe (it's additive, doesn't affect anything
existing). Renaming or removing an existing setting `name` that templates already
reference will silently break those references — grep the whole theme for a setting name
before renaming or deleting it.

## JavaScript conventions

Two scripts nearly every theme loads: the theme's own `assets/js/application.js`
(carousels, basket AJAX, product galleries, modals, quick view, form validation — theme-
specific, varies a lot between themes) and a shared, externally-hosted
`plugins.min.js` (`https://s3-eu-west-1.amazonaws.com/shopwired-theme-assets/v3/js/plugins.min.js`)
providing common jQuery plugins used across most ShopWired themes. Both are typically
loaded via `html.script(...)` in `master.twig`'s scripts block.

**Foundation Sites** (the CSS/JS framework, loaded from a CDN — check `master.twig`'s
stylesheets/scripts blocks for the version) provides most of the interactive-chrome
plumbing:

- **Reveal** (`data-reveal`, `data-open="targetId"`, `data-close`) — full modal dialogs
  with a backdrop, scroll lock, and open/close events (`open.zf.reveal`/`closed.zf.reveal`)
  handled for you.
- **Toggler** (`data-toggler=".some-class"` on the target, `data-toggle="targetId"` on any
  trigger) — a much lighter mechanism that just adds/removes a class on the target; no
  backdrop, no scroll lock, no focus trapping. Events fire as `on.zf.toggler` (class just
  got *added*) / `off.zf.toggler` (class just got *removed*) — note the naming is relative
  to the toggled class, not to visibility, so if the toggled class is `.hide`, `off.zf.toggler`
  means the element just became *visible*. If you need a backdrop/scroll-lock with
  Toggler (Reveal doesn't give you these), hand-roll them: toggle a `body` class in the
  `on`/`off` handlers and give that class a CSS `::before` pseudo-element backdrop plus
  `overflow: hidden`.
- **Equalizer** (`data-equalizer="<group-id>"` + `data-equalize-by-row="true"` on a
  container) — equalizes the height of matching child elements. Re-run it after
  dynamically injecting content by triggering a `resize` event on `window`
  (`$(window).trigger('resize')`) rather than trying to call the plugin API directly.

**Before writing new JS for a common pattern (a search box, a modal, a slider), grep
`application.js` for handlers that already reference the element ID/class you're about to
introduce.** It's common in this theme family for a shared base `application.js` to ship
with dormant handlers for functionality a given theme instance hasn't wired up its markup
for yet (e.g. a `$('#header-search').on('off.zf.toggler', ...)` focus handler sitting
unused because no element with that ID exists in the theme's current `header.twig`) —
reuse these rather than duplicating the behavior under a different selector.

## End-to-end workflow checklist for a change

1. Confirm you have a valid `config.json` for the *correct* theme before doing anything
   else — if there's any ambiguity, do a cheap round-trip verification (download one known
   file, diff against what's expected) before trusting further operations.
2. Read the actual current files before editing — themes drift from what you might assume
   based on a similar theme or an old memory of the codebase.
3. Make the edit.
4. Validate locally: `node --check` any touched `.js`, standalone-compile any touched
   `.scss` partial with `sass`.
5. Upload only the files you actually changed (explicit paths, not the whole theme,
   unless that's genuinely what's needed).
6. Verify at least one upload landed with a round-trip diff, especially the first upload
   in a session or after any credential change.
7. If the project also tracks the theme in git (recommended, since the CLI has no
   history/diff/rollback of its own beyond ShopWired's own "revert to previous version"
   feature), commit only when the user asks you to — but do keep git and the live theme in
   sync; don't let them silently diverge across a long session.
