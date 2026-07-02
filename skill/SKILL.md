---
name: tabfs
description: >-
  Read and act on the user's REAL, running browser through the ~/chrome
  filesystem mount (TabFS): list and triage open tabs, read a page's
  URL/title/text/HTML, capture a tab's console output, screenshot a window,
  run JS in a page, and read/write bookmarks — all as ordinary file
  operations (cat, ls, grep, echo, rm). Use whenever a task concerns the
  user's actual open tabs, bookmarks, or windows rather than fetching a URL
  fresh — e.g. "what do I have open about X", "close the duplicate tabs",
  "grab the text of the page I'm looking at", "what's erroring in the
  console", "bookmark these". Assumes the TabFS mount is present; run the
  bundled `doctor` first if anything seems off.
---

# TabFS — the live browser as a filesystem

The user's running browser is mounted at **`~/chrome`** (a symlink to a FUSE
mount). Every open tab, bookmark, and window is a file or folder you operate on
with plain shell tools. This is the user's *actual* session — real logins, real
cookies, real tabs — not a clean automation browser. Act accordingly.

## Before you start

Run the bundled check once per session if there's any doubt:

```sh
bash "$CLAUDE_SKILL_DIR/doctor"     # or: ./doctor  from this skill's folder
```

It verifies the mount is present, **responsive** (a hung mount blocks forever),
and wired to the extension. If it reports a hang, the user must reload the TabFS
extension in the browser — you cannot fix that from the shell.

## Two rules that matter most

1. **Never close, delete, or overwrite the user's browser state without asking.**
   Closing a tab (`echo close > .../control`, `rm` a tab) or removing a bookmark
   is destructive and often irreversible. Triage and propose; let the user
   confirm. (Some users keep their own conventions — e.g. "tabs are TODOs,
   bookmarks are the archive" — in their CLAUDE.md/memory; respect those.)

2. **Never read every tab's content at once.** There can be *hundreds* of tabs.
   `cat`-ing all `text.txt`/`body.html` loads every page into memory. Always
   **filter by `url.txt`/`title.txt` first**, then read content only for the
   few tabs that matter.

   ```sh
   # find, don't slurp:
   grep -l example.com ~/chrome/tabs/by-id/*/url.txt
   ```

## Map

```
~/chrome/
├─ tabs/
│  ├─ by-id/<id>/            one folder per open tab
│  │  ├─ url.txt title.txt   metadata — always cheap to read
│  │  ├─ text.txt body.html  LIVE page content (innerText / innerHTML)
│  │  ├─ active              "true"/"false"; echo true > to focus the tab
│  │  ├─ control             echo close|reload|goBack|goForward|discard >
│  │  ├─ console             page console.* + uncaught exceptions (see below)
│  │  ├─ evals/x.js          echo JS > to run it; read x.js.result for output
│  │  ├─ inputs/<id>.txt     read/write the value of a text input on the page
│  │  ├─ watches/<expr>      touch a file named after a JS expr; cat re-evals
│  │  ├─ window              symlink to the tab's window
│  │  └─ debugger/           resources/ and scripts/ (attaches CDP — banner!)
│  ├─ by-title/  by-window/  the same tabs, indexed differently
│  ├─ last-focused           symlink to the tab the user is looking at
│  └─ create                 echo a URL > to open a new tab
├─ bookmarks/                the bookmark tree as folders + files
│  ├─ Bookmarks_bar/ Other_bookmarks/ Mobile_bookmarks/
│  │     a folder = a bookmark folder; a file = a bookmark (contents: its URL)
│  └─ by-id/<id>/            flat access: title.txt, url.txt
├─ windows/<id>/
│  ├─ tabs/                  this window's tabs
│  ├─ focused               "true"/"false"
│  ├─ visible-tab.png       screenshot of this window's active tab
│  └─ create                echo a URL > to open a tab in this window
├─ extensions/<name>.<id>/enabled   echo true|false > to toggle an extension
└─ runtime/
   ├─ reload                echo > to reload the TabFS extension
   └─ routes.html           full, authoritative route reference
```

## Recipes

```sh
# What am I looking at right now?
cat ~/chrome/tabs/last-focused/title.txt
cat ~/chrome/tabs/last-focused/url.txt

# Triage: list every open tab as "id  url"
for d in ~/chrome/tabs/by-id/*/; do printf '%s  %s\n' "$(basename "$d")" "$(cat "$d/url.txt")"; done

# Read the text of one specific page (only after narrowing by url)
cat ~/chrome/tabs/by-id/1234/text.txt

# Run JS in a page and get the result
echo 'document.querySelectorAll("a").length' > ~/chrome/tabs/by-id/1234/evals/count.js
cat ~/chrome/tabs/by-id/1234/evals/count.js.result

# Screenshot the focused window
win=$(basename "$(readlink ~/chrome/tabs/last-focused/window)")
cp ~/chrome/windows/$win/visible-tab.png /tmp/shot.png

# Open a tab
echo 'https://example.com' > ~/chrome/tabs/create

# Bookmarks: search the archive, add one
grep -rl example.com ~/chrome/bookmarks/ 2>/dev/null
echo 'https://example.com' > ~/chrome/bookmarks/Bookmarks_bar/Example
```

### Console capture

`cat ~/chrome/tabs/by-id/<id>/console` shows that tab's `console.*` output and
uncaught exceptions. The first read attaches the debugger and replays the page's
console history; it then keeps capturing, so later reads pick up new messages.
`: > console` clears the buffer.

```sh
cat ~/chrome/tabs/by-id/1234/console        # history + everything since
grep -i error ~/chrome/tabs/by-id/1234/console
: > ~/chrome/tabs/by-id/1234/console        # clear
```

Note: reading `console` (or anything under `debugger/`) attaches Chrome's
DevTools protocol, which shows a **"TabFS started debugging this browser"**
banner on that tab. For `console` the banner is bounded: TabFS detaches (and the
banner clears) as soon as the tab **navigates to a new page**, so it never rides
along to a page you didn't read. `debugger/` routes stay attached until the
extension reloads. If you have DevTools open on a tab, reading its `console`
fails (EIO) rather than stealing your DevTools session — close DevTools first.

## Gotchas

- **Lazily-restored / discarded tabs throw I/O errors** on `text.txt`,
  `body.html`, `console`, etc. until the user actually visits them. `url.txt`
  and `title.txt` still work. Treat an I/O error on content as "tab not loaded,"
  not a bug.
- **`chrome://`, the Web Store, other extensions, `view-source:`** can't be
  scripted — reading their content fails by design (not in `<all_urls>`).
- **Writes ignore truncation:** `echo X > file` first truncates (empty write,
  ignored) then writes X — so a URL/title isn't clobbered to empty.
- **If the mount hangs or errors everywhere:** the user must reload the TabFS
  extension in the browser. Say so; don't keep retrying.
- **`runtime/routes.html`** is the ground truth for every route and its usage —
  consult it when this map is not enough.
