# How macOS Quick Look Works, and How to Diagnose It

> This document is a deeper investigation into how macOS Quick Look resolves file previews. It is broader than what this project specifically addresses — it was written while diagnosing a separate case (a backup file with a `.py~` extension that gets a synthetic `dyn.…` UTI) — but the architectural model and the diagnostic commands here apply to any "why doesn't this file preview?" question, including the YAML and TOML cases this project's extension fixes. If you are trying to understand *why* `easy-plaintext-quicklook-extension` is structured the way it is, or you want to add Quick Look support for a format whose UTI behaves unexpectedly, start here.

This is a write-up of an investigation into why three files behave differently when previewed with the spacebar in Finder on macOS 26.3.1 (Tahoe):

| File | Quick Look result |
|---|---|
| `~/Projects/home-assistant-auto-updater/readme.md` | Renders as formatted Markdown |
| `~/Backup/rpi43/home/pi/button-shutdown.py` | Renders as syntax-aware text |
| `~/Backup/rpi43/home/pi/button-shutdown.py~` | Shows the generic "page with question mark" icon |

## 1. How Quick Look works under the hood

Quick Look is a system service (`quicklookd`) that Finder, Spotlight, Mail, and other apps call into to render a preview for a file. Resolving "what should I show for this file?" happens in three layers.

### Layer A — Launch Services assigns a UTI

When a file appears on disk, Launch Services maps its filename extension (and sometimes the bytes inside it) to a Uniform Type Identifier, or UTI. A UTI is a reverse-DNS string like `net.daringfireball.markdown` or `public.python-script`. Each UTI declares which other UTIs it conforms to (for example, Markdown conforms to `public.plain-text`, which conforms to `public.text`, which conforms to `public.data`). The set of conformances is the file's content-type tree, and most Quick Look behavior is driven by it rather than by the leaf UTI alone.

If Launch Services has no registration that matches the extension, it does not give up — it synthesizes a placeholder UTI of the form `dyn.<base32-encoded-extension>`. A dynamic UTI conforms only to `public.item` and `public.data`. It is, by design, a "we have no idea what this is" sentinel.

### Layer B — A Quick Look provider claims one or more UTIs

Two generations of providers coexist:

1. **Old-style `.qlgenerator` bundles** (the original Quick Look API) live in `/Library/QuickLook/`, `~/Library/QuickLook/`, `/System/Library/QuickLook/`, or inside an app bundle. Each generator declares a list of UTIs it can render. This is the API third-party tools like QLMarkdown and Peek used.
2. **Modern `.appex` Preview Extensions** (introduced with the App Extensions framework) ship inside an app bundle under `Contents/PlugIns/...appex` and register themselves with `pluginkit` at the extension point `com.apple.quicklook.preview`. Their `Info.plist` lists `QLSupportedContentTypes`. This is how Apple ships its own previewers today.

When asked to preview a file, `quicklookd` walks the file's content-type tree (leaf first, then ancestors) and picks the first provider that claims any UTI in the tree.

### Layer C — The provider returns a rendered view

Old generators return a thumbnail or HTML/PDF preview. Modern preview extensions return a view controller. Either way the calling app displays the result inside the Quick Look panel.

The practical consequence of this design is the central diagnostic question for any "why doesn't Quick Look work?" issue: **does the file's UTI tree contain any UTI that some installed provider claims?** If yes, you get a preview. If no — most often because the file got a `dyn.…` UTI — you get the question-mark icon.

## 2. How to diagnose a specific file

Three commands do almost all of the work.

### Step 1 — Ask what UTI Launch Services assigned

```
mdls -name kMDItemContentType -name kMDItemContentTypeTree -name kMDItemKind <file>
```

This is the first thing to check, because everything downstream depends on the answer. The interesting field is `kMDItemContentTypeTree` — the leaf UTI plus all of its ancestors.

### Step 2 — List every Quick Look provider that exists on the system

For old-style generators:

```
qlmanage -m plugins
```

For modern preview extensions:

```
pluginkit -mAvvv -p com.apple.quicklook.preview
```

The first lists `<UTI> -> <generator path>` mappings directly. The second lists each extension's bundle and path; to see which UTIs a given extension actually claims, read the `QLSupportedContentTypes` array in its `Info.plist`:

```
/usr/libexec/PlistBuddy -c "Print" <appex>/Contents/Info.plist | grep -A 20 NSExtension
```

### Step 3 — Cross-reference

For each UTI in the file's content-type tree, ask whether any provider from Step 2 claims it. The first hit (leaf-first) is what `quicklookd` will use. If nothing in the tree is claimed, Finder falls back to the generic icon.

## 3. Applying this to the three example files

### File 1 — `readme.md` (works)

```
$ mdls -name kMDItemContentType -name kMDItemContentTypeTree -name kMDItemKind \
    /Users/justin/Projects/home-assistant-auto-updater/readme.md
kMDItemContentType     = "net.daringfireball.markdown"
kMDItemContentTypeTree = (
    "net.daringfireball.markdown",
    "public.plain-text",
    "public.text",
    "public.data",
    "public.item",
    "public.content"
)
kMDItemKind            = "Markdown Document"
```

Launch Services recognized the `.md` extension and assigned the well-known Markdown UTI. The tree includes `public.plain-text`. Now look for a provider that claims any of these:

```
$ qlmanage -m plugins | grep -E "markdown|plain-text"
  public.plain-text -> /System/Library/QuickLook/Text.qlgenerator (1018.3.2)
```

The legacy `Text.qlgenerator` claims `public.plain-text`, so a preview would have worked even on older macOS. But on macOS Sonoma and later there is also a system preview extension, which is what actually renders the formatted Markdown:

```
$ /usr/libexec/PlistBuddy -c "Print" \
    /System/Library/Frameworks/QuickLookUI.framework/Versions/A/PlugIns/QLPreviewGenerationExtension.appex/Contents/Info.plist \
    | grep -A 20 NSExtension
NSExtension = Dict {
    NSExtensionAttributes = Dict {
        QLIsDataBasedPreview = true
        QLSupportedContentTypes = Array {
            public.plain-text
            public.rtf
            com.apple.rtfd
            org.oasis-open.opendocument.text
            com.apple.property-list
            public.xml
            public.json
        }
    }
    NSExtensionPointIdentifier = com.apple.quicklook.preview
}
```

`QLPreviewGenerationExtension` is the OS-bundled previewer that handles `public.plain-text` and renders Markdown source as formatted output. Nothing was installed by hand to make this work. Both `/Library/QuickLook/` and `~/Library/QuickLook/` are empty on this machine, which confirms there is no third-party generator involved.

### File 2 — `button-shutdown.py` (works)

```
$ mdls -name kMDItemContentType -name kMDItemContentTypeTree -name kMDItemKind \
    /Users/justin/Backup/rpi43/home/pi/button-shutdown.py
kMDItemContentType     = "public.python-script"
kMDItemContentTypeTree = (
    "public.python-script",
    "public.shell-script",
    "public.script",
    "public.source-code",
    "public.plain-text",
    "public.text",
    "public.data",
    "public.item",
    "public.content"
)
kMDItemKind            = "Python Script"
```

The `.py` extension is registered to `public.python-script`, and that UTI conforms (eventually) to `public.plain-text`. The same `QLPreviewGenerationExtension` from above claims `public.plain-text`, so it handles this file too. The legacy `Text.qlgenerator` is also a fallback for the same UTI.

### File 3 — `button-shutdown.py~` (broken)

```
$ mdls -name kMDItemContentType -name kMDItemContentTypeTree -name kMDItemKind \
    /Users/justin/Backup/rpi43/home/pi/button-shutdown.py~
kMDItemContentType     = "dyn.ah62d4rv4ge81a8p8"
kMDItemContentTypeTree = (
    "public.item",
    "dyn.ah62d4rv4ge81a8p8",
    "public.data"
)
kMDItemKind            = "Document"
```

This is the full explanation. Launch Services has no registration for the `.py~` extension (Emacs's autosave/backup convention is not a UTI Apple knows about), so it minted a synthetic dynamic UTI. A dynamic UTI's tree contains only `public.item` and `public.data` — not `public.plain-text`, not `public.text`, not `public.source-code`. No Quick Look provider on the system claims `public.item` or `public.data`, so `quicklookd` has nothing to invoke and Finder shows its generic "unknown document" icon.

The contents of the file are fine — they are just plain Python source. The problem is purely that the `.py~` extension is not associated with a textual UTI.

## 4. Fixing the `.py~` case

There is no point installing a "py~ Quick Look plugin" — none exists, and the underlying issue isn't a missing renderer but a missing UTI association. Two reasonable fixes:

1. **Rename or strip the trailing `~` on a per-file basis.** Once the file ends in `.py`, Launch Services classifies it as `public.python-script` and the existing previewer handles it.
2. **Teach Launch Services that `.py~` is plain text.** Create a small helper app whose `Info.plist` exports a UTI declaration for the `py~` extension that conforms to `public.plain-text`, then register it (`lsregister -f /path/to/App.app`). After that, `mdls` on a `.py~` file will report a real UTI tree containing `public.plain-text` and Quick Look will work without renaming.

Option 1 is one-line and stateless; option 2 is the right fix if you have a directory full of these and don't want to rename them.

> Option 2 is conceptually similar to what `easy-plaintext-quicklook-extension` does, but in the opposite direction. This project does **not** export new UTI declarations; it accepts the existing UTI Launch Services assigns (e.g. `public.yaml`, `public.toml`) and adds a *Quick Look provider* that claims those UTIs. Both approaches solve the "no provider in this file's tree" problem — one by changing the tree, the other by changing the provider list.

## 5. Cheat sheet

```
# What does macOS think this file is?
mdls -name kMDItemContentType -name kMDItemContentTypeTree <file>

# Which old-style generators are installed, and what do they claim?
qlmanage -m plugins

# Which modern preview extensions are installed?
pluginkit -mAvvv -p com.apple.quicklook.preview

# What UTIs does a specific extension actually handle?
/usr/libexec/PlistBuddy -c "Print" <App>.app/Contents/PlugIns/<Ext>.appex/Contents/Info.plist \
    | grep -A 20 NSExtension

# Render a preview interactively (opens a real Quick Look window titled
# "[DEBUG] <filename>" — qlmanage takes over the foreground briefly):
qlmanage -p <file>

# Same idea, but force a specific UTI to test "would this work if Launch Services
# classified the file differently?" — useful for diagnosing dyn.… cases:
qlmanage -p -c public.python-script <file>

# Render a thumbnail to a PNG on disk instead of opening a window. Quieter, scriptable,
# and the presence/absence of an output file is itself diagnostic — if no PNG appears,
# no provider claimed any UTI in the file's tree:
qlmanage -t -s 800 -o /tmp/ql-test <file>
```

Two practical notes about `qlmanage -t`:

- It writes one `<basename>.png` per input file into the output directory. On the three example files, `readme.md.png` and `button-shutdown.py.png` appeared immediately; nothing was written for `button-shutdown.py~`.
- When given a file with a `dyn.…` UTI, the command does not fail fast — it hangs silently waiting for a provider that will never be found. If a thumbnail does not appear within a couple of seconds, that is the answer; kill the process.

A `dyn.…` UTI in the output of `mdls` is almost always the root cause of a missing Quick Look preview, and is the single most useful signal to look for.
