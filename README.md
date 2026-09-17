# Safari runs a module worker's entry file twice

A two-file reproduction of a WebKit bug: in Safari, any import of a module
worker's own entry file evaluates it a **second** time, creating a separate
module instance with its own state. Chrome and Firefox return the module that
is already running, which is what the HTML Standard requires.

- Write-up: [Safari runs a module worker's entry file twice if anything imports it](https://dev.to/tom_limb/safari-runs-a-module-workers-entry-file-twice-if-anything-imports-it-dbp)
- WebKit bug: [324459](https://bugs.webkit.org/show_bug.cgi?id=324459)

## Run it

Module workers don't load from `file://`, so serve the folder:

```sh
python3 -m http.server
```

Then open http://localhost:8000 in Safari, and again in Chrome or Firefox.

| Browser | Result |
| --- | --- |
| Safari 26.5 | `{"evals":2,"same":false}` — the entry ran twice, and `helper.js` holds the second copy |
| Chrome 153 | `{"evals":1,"same":true}` |
| Firefox 156 | `{"evals":1,"same":true}` |

## What it does

`w.js` is the worker's entry. It imports `helper.js`, which imports `w.js`
back — an ordinary static import cycle, no bundler and no dynamic import.
`evals` counts how many times the entry was evaluated, and `same` reports
whether `helper.js` sees the same module instance the worker is running.

## Why it matters

Bundlers put code shared between a worker's entry and its lazily loaded chunks
into the entry chunk, and the chunks import it back from there. In Safari that
silently duplicates module-level state — registries, caches, singletons,
"already initialised" flags — and re-runs top-level side effects. Nothing
throws. It's how a WASM decoder registry ended up empty in Safari only, in the
tools at [ohsovideo.com](https://ohsovideo.com).

Two things avoid it:

1. Move shared code out of the worker's entry into its own chunk, so nothing
   imports the entry (with Rollup, `output.manualChunks`).
2. Make the entry a thin loader: `import("./main.js")` and nothing else.

## Per the spec

Each worker has [one module map](https://html.spec.whatwg.org/multipage/workers.html#run-a-worker),
and the worker's top-level script is fetched through it, keyed by URL. The
[module map](https://html.spec.whatwg.org/multipage/webappapis.html#module-map)
exists "to ensure that imported module scripts are only fetched, parsed, and
evaluated once per Document or worker."

(One caveat: the key is the exact URL. A worker started as `w.js?v=2` and
imported later as `w.js` is legitimately two modules.)

## Licence

Public domain (CC0). Copy it into a bug report or a test suite freely.
