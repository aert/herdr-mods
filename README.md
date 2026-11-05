# herdr-app: Files Transfer + Preview Plus, stacked

For an agent, on a machine where Herdr is installed. Both mods carry their own
documentation (menus, editing, limits, licences) — this file is only the wiring:
what to fetch, where each mod folder goes, and the exact Files-panel stack, which is
the one part the two upstream READMEs state differently.

## The two mods, and why these

| mod | source | what it brings |
| --- | --- | --- |
| **Files Transfer** | [vladimiryakimenko/herdr-files-transfer](https://github.com/vladimiryakimenko/herdr-files-transfer) · Apache-2.0 | Herdr's Files panel plus transfer: right-click download a file or a whole folder to `~/Downloads`, upload by paste or drag, new file/folder, rename, Trash (delete on another machine, after asking), copy path, reveal in Finder, a VS Code icon set |
| **Preview Plus** | [linziyanleo/herdr-mods](https://github.com/linziyanleo/herdr-mods) → [`preview-plus`](https://github.com/linziyanleo/herdr-mods/tree/main/preview-plus) · no licence file of its own | formatted Markdown with followable relative links, images fitted and zoomable, PDF pages, Preview/Source toggle, edit and save |

Why not something built here: **Herdr has no way to pull a file off another machine.**
Its own [`files` mod](https://github.com/herdrdev/herdr-app/blob/master/defaults/mods/files/main.js)
is a read-only browser — listing, preview, git letters — and everything the app does
with files across machines goes the other way: ⌘V / `terminal.paste_image` push a
copied file or image *into* a terminal, `terminal.put` writes bytes beside one, and
`hc file read` / `hc file write` are text-only and capped at 8 MB. Files Transfer
fills exactly that gap, copying with `scp` over the same ssh address Herdr already
uses for the machine. Preview Plus is its pair — Files Transfer hands it Markdown,
images and PDFs, and keeps the source view for every other file — which is why the
panel is a *stack* of the two rather than two panels: the tree keeps one identity
(`files`) and one file in view.

Not this one: [kosuketut/herdr-remotedownloder](https://github.com/kosuketut/herdr-remotedownloder)
also downloads remote files to the Mac, but it is a plugin for the older `herdr`
terminal multiplexer (`herdr plugin install`, `prefix+d`, "Herdr 0.7.0 or later") —
a different program from the `herdr-app` desktop app this file is about.

Both are third-party mods, listed in the [herdr.dev gallery](https://herdr.dev/mods/),
and neither is pinned here.

## Optional: Reviewer, in the Changes slot

[vytasgavelis/herdr-reviewer](https://github.com/vytasgavelis/herdr-reviewer) · MIT.
Replaces the Changes panel's diff with a pull-request-style review of the working
tree: syntax-coloured diffs, three scopes (uncommitted, last commit, vs main),
inline comments on lines, hunks or files, and *Submit review* sends them all to
the agent in the space. It keeps the file list and commit box. Independent of the
Files stack: it takes the `changes` child of `panel-body`, not `files`. Default
branch is `master`.

```sh
w2=$(mktemp -d)
curl -fsSL https://github.com/vytasgavelis/herdr-reviewer/archive/refs/heads/master.tar.gz | tar xz -C "$w2"
sh "$w2/herdr-reviewer-master/scripts/install.sh"   # copies to $app/mods/reviewer and checks it
```

Then, in each layout you use (not Herdr's read-only `main`; clone it first), point
the `"id": "changes"` block at it:

```json
{ "id": "changes", "type": "block", "content": { "mod": "reviewer" } }
```

No keys are bound by default (`reviewer.submit`, `reviewer.ask`, `reviewer.clear`,
`reviewer.scope` can be bound in `keys.json`).

```sh
set -eu
app=${HERDR_APP_HOME:-$HOME/.config/herdr-app}
[ -d "$app/mods" ] || { echo "start Herdr once first: no $app"; exit 1; }
if [ -e "$app/mods/files-transfer" ] || [ -e "$app/mods/preview-plus" ]; then
  echo "already installed — ask the person before replacing"
fi

w=$(mktemp -d)
curl -fsSL https://github.com/vladimiryakimenko/herdr-files-transfer/archive/refs/heads/main.tar.gz | tar xz -C "$w"
curl -fsSL https://github.com/linziyanleo/herdr-mods/archive/refs/heads/main.tar.gz | tar xz -C "$w"
cp -R "$w/herdr-files-transfer-main/mods/files-transfer" "$app/mods/"
cp -R "$w/herdr-mods-main/preview-plus/mods/preview-plus" "$app/mods/"

herdr-app check "$app/mods/files-transfer"   # expect: check passed
herdr-app check "$app/mods/preview-plus"     # expect: check passed
```

A shipped layout is read-only, so let Files Transfer clone the one on screen and
claim its Files block:

```sh
sh "$w/herdr-files-transfer-main/install.sh"
```

Then, in `$app/layouts/<name>-files-transfer.json`, replace the `files` block with
this stack (keep the surrounding `panel-body` stack, its `changes` child, the
`panel` column's size and visibility, and any `size`/`min`/`max`/`hidden` the old
block carried):

```json
{
  "id": "files",
  "type": "stack",
  "active": "files-plus-tree",
  "children": [
    { "id": "files-plus-tree", "type": "block", "content": { "mod": "files-transfer", "props": { "previewStack": "files" } } },
    { "id": "preview-plus",     "type": "block", "content": { "mod": "preview-plus",   "props": { "previewStack": "files" } } }
  ]
}
```

Load-bearing, because that is how the two mods hand files to each other: the stack's
`id` stays `files` (the name `props.previewStack` gives), both blocks set
`"previewStack": "files"`, and the child ids are exactly `files-plus-tree` and
`preview-plus` (the child `files-transfer` selects).

Verify:

```sh
herdr-app status                                        # "errors": [] and the new layout
herdr-app do layout.select --stack files --child preview-plus     # panel: "Select a file in Files Plus."
herdr-app do layout.select --stack files --child files-plus-tree  # back to the tree (what ← runs)
```

Then in the window: right-click a file → **Download** lands it in `~/Downloads`;
click a `.md`/image/PDF → Preview Plus renders it and ← returns. For a file on
another machine, the download goes over `scp` to the `target` `hc machine ls --json`
reports; if it fails, `scp -o BatchMode=yes <target>:/etc/hostname /tmp/` is the test.

Requirements: macOS for Files Transfer's paste/copy-path/reveal/Trash and for Preview
Plus's PDF pages; remote machines only need to be ssh targets with keys. Invalid JSON
leaves the last good layout and writes `$app/layouts/<name>.error`; an empty panel
means the focused pane has no folder yet. Updating and uninstalling are re-copying and
deleting those two folders and the cloned layout file — the upstream READMEs have the
rest.
