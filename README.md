# TabFS

Mount your running browser as a filesystem: every open tab, bookmark, and
window becomes a file you can read and write with plain shell tools.

This is a fork of [Omar Rizwan's TabFS](https://omar.website/tabfs/)
([osnr/TabFS](https://github.com/osnr/TabFS)), updated and extended for use by
AI coding agents.

## Usage

```sh
$ ls ~/chrome/
bookmarks  extensions  history  runtime  tabs  windows

$ cat ~/chrome/tabs/last-focused/title.txt        # the tab you're looking at
$ grep -l github.com ~/chrome/tabs/by-id/*/url.txt # find tabs by URL
$ echo 'https://example.com' > ~/chrome/tabs/create # open a tab
$ echo reload > ~/chrome/tabs/by-id/<id>/control    # reload / remove / discard
$ echo 'document.title' > ~/chrome/tabs/by-id/<id>/evals/t.js
$ cat ~/chrome/tabs/by-id/<id>/evals/t.js.result    # run JS, read the result
$ cat ~/chrome/windows/<id>/visible-tab.png > shot.png # screenshot
$ cat ~/chrome/history/search/kubernetes            # free-text history search
```

`~/chrome/runtime/routes.html` documents every route.

Everything is computed on demand — no content scripts, no polling — so the
mount costs the browser nothing while idle. The one exception is `console`,
which keeps a debugger session attached to that tab until it navigates.

## Fork additions

- Migration to Chrome's current extension protocol (Manifest V3)
- `bookmarks/` — the bookmark tree as folders and files, plus a flat `by-id/` view
- `history/` — `by-time` (recent visits, TSV, newest first) and `search/<text>`
- `tabs/by-id/<id>/console` — the page's console output and uncaught
  exceptions, captured through the debugger. When a console is read, Chrome
  shows a *«"TabFS" started debugging this browser»* banner: that is the sign
  it works, and you can dismiss it
- `skill/` — a [Claude Code](https://claude.com/claude-code) skill teaching an
  agent the layout and the house rules, with a `doctor` script that checks the
  mount end to end

Everything else — per-tab `text.txt`/`body.html`, `control`, `evals/`,
`inputs/`, `watches/`, window screenshots — is upstream TabFS.

## Install

1. Build the FUSE daemon: `cd fs; make` (requires libfuse).
2. Load `extension/` as an unpacked extension (`chrome://extensions`,
   Developer mode, *Load unpacked*) and note its ID.
3. Register the native messaging host: `./install.sh chrome <EXTENSION_ID>`.
4. The browser appears under `fs/mnt/`; symlink it somewhere comfortable:
   `ln -s "$(pwd)/fs/mnt" ~/chrome`.

See the [upstream instructions](https://omar.website/tabfs/) for other
browsers and platforms.

## Security

This exposes your browsing content — tabs, history, bookmarks, live pages — to
whatever reads the mount, including an AI agent. It trusts that reader to be
non-malicious and prudent. Point it at a browser profile you are comfortable
sharing.
