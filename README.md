# dsh-ipynb-preview

[![CI](https://github.com/WuShichao/dsh-ipynb-preview/actions/workflows/ci.yml/badge.svg)](https://github.com/WuShichao/dsh-ipynb-preview/actions/workflows/ci.yml)
[![npm](https://img.shields.io/npm/v/dsh-ipynb-preview)](https://www.npmjs.com/package/dsh-ipynb-preview)
[![downloads](https://img.shields.io/npm/dm/dsh-ipynb-preview)](https://www.npmjs.com/package/dsh-ipynb-preview)
[![license](https://img.shields.io/npm/l/dsh-ipynb-preview)](LICENSE)
[![dsh-plugin](https://img.shields.io/badge/topic-dsh--plugin-4D6BFE)](https://github.com/topics/dsh-plugin)

A [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) plugin that
renders Jupyter notebooks in the document preview pane — and, when
[dsh-better-sidebar](https://github.com/omdsh-dev/DSH-better-sidebar) is
installed, in that plugin's sidebar file viewer too: syntax-highlighted code,
real LaTeX, and zoomable figures — instead of raw JSON.

Without it, opening a `.ipynb` in DSH shows its JSON source. With it, the file
opens as a notebook.

| Feature | Notes |
|---|---|
| Syntax highlighting | Python, JavaScript/TypeScript, JSON, Bash, and INI, with per-language grammars |
| LaTeX math | Vendored KaTeX, so it works offline; inline `$…$` and display `$$…$$`, including inside tables and lists |
| Markdown subset | Headings, lists, tables with column alignment, blockquotes, task lists, fenced code, links, bold and italic, code spans |
| Figure zoom | Click any output figure to open it, then scroll or `+` / `−` to zoom continuously, drag to pan, `0` to fit |
| Save figures | Names each figure after the notebook, e.g. `sample-notebook-cell3-out1.png` |
| Outputs | Streams, `display_data`, `execute_result`, and error tracebacks with ANSI escapes stripped |
| Sidebar preview | With `dsh-better-sidebar`, a notebook opened in its file tree or editor renders as a notebook, not as JSON in the code viewer |

Everything is self-contained: the KaTeX engine and its web fonts are inlined into
the client bundle, so no request leaves your machine.

## Install

```sh
dsh plugin --profile <your-profile> add dsh-ipynb-preview
```

Then restart DSH. The plugin is a client-side renderer, so it needs a restart
rather than a page refresh: the shell computes its client bundle revision at
startup.

To install from a checkout or a git URL instead:

```sh
dsh plugin --profile <your-profile> add ./dsh-ipynb-preview
dsh plugin --profile <your-profile> add github:WuShichao/dsh-ipynb-preview
```

The git form makes pnpm ask for permission to run this package's `prepare`
script, because pnpm >= 10 refuses a git dependency's install-time script until
it is allowlisted. The built `lib/client.js` is committed, and the npm package
ships it too, so neither a tarball nor a registry install asks for anything.

## How it works

DSH discovers the package through the profile's bundle list, reads `dsh.client`
from this manifest, and serves `lib/client.js` to the browser as part of the
client roster. That bundle registers a document-preview renderer for the `ipynb`
extension:

```
lib/index.js           host half; deliberately empty, the plugin is browser-only
lib/client.template.js the editable client source, with two KaTeX placeholders
lib/client.js          the generated artifact the shell serves
cordis.patch.yml       the bundle layer that inserts the plugin row
tools/build-client.mjs generates lib/client.js from the template
```

`lib/client.js` is generated, not hand-edited. Change `lib/client.template.js`
and run `npm run build`.

## dsh-better-sidebar: notebooks in the sidebar

[dsh-better-sidebar](https://github.com/omdsh-dev/DSH-better-sidebar) takes over
the right sidebar's file tree and editor pane. A notebook opened there never
reaches the shipped document preview this plugin registers into, so without an
integration it falls to that plugin's catch-all code viewer — raw JSON with a
CodeMirror.

When better-sidebar is installed, this plugin therefore also registers a **file
viewer** for `.ipynb` through its `ctx.betterSidebar` service:

```js
scoped.effect(() => scoped.get("betterSidebar").registerFileViewer({
  id: "dsh-ipynb-preview:notebook",
  exts: ["ipynb"],
  title: () => "Jupyter notebook",
  priority: 0,
  fetchStrategy: "fsRead",
  component: BetterSidebarViewer,
}))
```

How it behaves:

- **Optional.** `betterSidebar` is deliberately *not* in the plugin's `inject`
  list: a profile without better-sidebar must keep the shipped preview. The
  registration runs in a scoped `ctx.inject(["betterSidebar"], …)` child, which
  never runs when the service is absent. Nothing else in the plugin changes.
- **No extra dependency.** No value or type import of `dsh-better-sidebar` is
  made, so it need not be installed at build time, and there is no second copy
  of it. The integration only needs the service (`registerFileViewer`), which
  exists since better-sidebar v0.4.0.
- **One renderer.** The sidebar viewer and the document preview render the same
  component, so the same notebook looks the same in both places; the test suite
  asserts both renders are byte-identical.
- **It scrolls.** better-sidebar mounts a viewer as a flex item of a column pane
  whose ancestor clips overflow, so the viewer root is its own scroll container
  (`flex: 1; min-height: 0; overflow: auto` — the contract its built-in viewers
  follow). The notebook itself stays a plain block, so the document preview
  keeps its single, host-owned scroller.
- **Wrapped source lines.** The sidebar pane is narrow, so code cells wrap there
  instead of scrolling sideways.
- **Read limit.** The sidebar hands a viewer at most 5 MB of text. A larger
  notebook shows its size and the limit instead of a render; the document
  preview has no such limit.
- **No network.** The sidebar reads the file through its own host route; the
  notebook still renders entirely in the browser.

Opening a notebook from a chat link, the workspace file list, or the sidebar
file tree all end up in the same renderer.

## Development

```sh
pnpm install    # installs KaTeX (build), React (tests)
pnpm run build  # regenerate lib/client.js
pnpm test       # build, then the verifier, the bundle lint, and the mount harness
```

The repository ships a `pnpm-lock.yaml`, and pnpm is what the DSH CLI itself uses
to install plugins, so CI installs with `pnpm install --frozen-lockfile`. npm and
yarn also work: every script is a plain `node` invocation rather than a chained
`npm run`, so no script depends on which package manager started it.

`pnpm test` runs three suites, all offline:

- **`tests/verify.mjs`** — materializes the bundle the way the shell's module
  loader does, then checks the manifest contract, the registrations, the
  highlighter, the math renderer, the Markdown subset, the zoom arithmetic, the
  stylesheet, and a full render of the fixture notebook. Around 170 assertions.
- **`tests/lint-bundle.mjs`** — sweeps both the template and the built bundle for
  identifiers that are referenced but never declared. A free variable makes the
  factory throw while the shell materializes it, and the whole client roster then
  fails to load — a blank window that `node --check` cannot detect.
- **`tests/mount.mjs`** — mounts the plugin against a DOM stub so effects and
  event wiring actually run, and asserts the notebook tree survives opening the
  lightbox.

The fixture notebook is generated by `tests/make-fixture.mjs`, and the assertions
read their expected counts from that fixture, so the input and the expectation
cannot drift apart.

### Interaction testing in a real browser

React's synthetic event system does not work against a hand-written DOM stub, so
the controls' click handlers are verified in a real browser instead:

```sh
node tests/make-page.mjs            # writes test-page.html at the repo root
node tests/make-page.mjs out.html   # or choose the output path
```

Open the page, click **Open the lightbox**, and press every control; each
interaction, the zoom readout, and any error is written to the log at the top of
the page. Append `?auto=1` to the URL and the page drives itself, which is how a
headless run reads the result back:

```sh
msedge --headless=new --virtual-time-budget=15000 --dump-dom \
  "file:///path/to/test-page.html?auto=1"
```

That page also hit-tests every control with `document.elementFromPoint`, which is
how a stacking-order bug was found: a fixed application header with a higher
`z-index` than the lightbox covered the middle of the toolbar, so only its edges
responded to a click.

## Requirements

- Node 18 or newer to build and test.
- React is **not** a dependency: the shell supplies it from its own frozen module
  table. It is a dev dependency only, for the tests.

## Notes for plugin authors

Two things this plugin learned the hard way, both now covered by assertions:

- A client bundle is materialized inside the shell's module loader. If the
  factory throws, the row has no exports and the whole roster fails — which looks
  like a blank window rather than a plugin error. Keep the factory free of
  undeclared identifiers.
- A full-screen overlay needs a `z-index` above every host layer. `9999` is not
  enough: application chrome is often a fixed header with its own stacking
  context, and it will silently swallow clicks aimed at your controls.

## License

MIT — see [LICENSE](LICENSE). KaTeX is bundled under its own MIT license; see
[THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).

Published to npm by [@nightwingng](https://www.npmjs.com/~nightwingng).
