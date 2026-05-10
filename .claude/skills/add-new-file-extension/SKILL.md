---
name: add-new-file-extension
description: Use this skill when the user wants to add Quick Look text-preview support for a new plain-text file extension (e.g. .toml, .py, .conf, .ini) to the easy-plaintext-quicklook-extension. Discovers the macOS UTI for a sample file, edits PreviewExtension/Info.plist to claim that UTI, rebuilds, reinstalls, and resets the Quick Look daemon. Usage: `/add-new-file-extension <path/to/sample.ext>`.
---

# /add-new-file-extension — extend coverage to a new file type

The user provides a path to a sample file (or a path is implicit from prior context). Run this sequence in order.

## 1. Pre-flight

Confirm the working directory contains `EasyPlaintextQuicklookExtension.xcodeproj/`. The extension must already be installed (run `/install` first if not).

## 2. Find the UTI

```sh
mdls -name kMDItemContentType -name kMDItemContentTypeTree <path-to-sample-file>
```

Parse the output. There are three cases:

**Case A — UTI macOS already knows.** `kMDItemContentType` is a recognizable string like `public.toml`, `public.python-script`, or `net.daringfireball.markdown`. Continue to step 3.

**Case B — UTI is dynamic (`dyn.…`).** macOS does not know the format. The user has two options:
  - Stop and ask whether to declare a custom UTI in `UTExportedTypeDeclarations`. This is more involved than the Case A path. If the user agrees, follow Apple's UTExportedTypeDeclarations docs to declare a custom UTI conforming to `public.plain-text` and bind the file extension via `UTTypeTagSpecification`. Then add the custom UTI to `QLSupportedContentTypes` (continue to step 3 with the custom UTI string).
  - Alternatively, suggest the user just register the extension with `public.plain-text` itself. This is a hack but works for files that are unambiguously plain text.

**Case C — UTI does not conform to `public.text`.** Inspect `kMDItemContentTypeTree`. If `public.text` is not in the list, the file is not actually a text format. The current extension hands bytes to macOS labeled as plain text and would render binary content as garbage. Stop and tell the user this format would need a different rendering strategy — this skill cannot help.

## 3. Check for duplicate

Read `PreviewExtension/Info.plist` and confirm the UTI is not already in `QLSupportedContentTypes`. If it is, tell the user the UTI is already supported and stop. If preview is broken despite the UTI being registered, the issue is elsewhere (run `/install` to re-register, or check the System Settings extensions toggle).

## 4. Edit Info.plist

Add the UTI as a new `<string>` entry inside the `QLSupportedContentTypes` array in `PreviewExtension/Info.plist`. Use the `Edit` tool with the existing array contents as `old_string` and the existing contents plus the new `<string>` entry as `new_string`.

Example: if the array currently contains `public.yaml` and you're adding `public.toml`:

```xml
<key>QLSupportedContentTypes</key>
<array>
    <string>public.yaml</string>
    <string>public.toml</string>
</array>
```

## 5. Rebuild

```sh
xcodebuild -project EasyPlaintextQuicklookExtension.xcodeproj \
           -scheme EasyPlaintextQuicklookExtension \
           -configuration Debug build 2>&1 | tail -3
```

Confirm `** BUILD SUCCEEDED **`.

## 6. Reinstall

```sh
SRC=$(xcodebuild -showBuildSettings -project EasyPlaintextQuicklookExtension.xcodeproj \
       -scheme EasyPlaintextQuicklookExtension -configuration Debug 2>/dev/null \
       | awk -F' = ' '/BUILT_PRODUCTS_DIR =/ {print $2}')/EasyPlaintextQuicklookExtension.app
DEST=/Applications/EasyPlaintextQuicklookExtension.app
[ -e "$DEST" ] && rm -rf "$DEST"
cp -R "$SRC" "$DEST"
open "$DEST"
sleep 2
killall QuickLookUIService 2>&1 || true
```

## 7. Tell the user to verify

> Done. Spacebar a file of the new type in Finder. The contents should now render as text.

Tell the user they can quit the host app's window (Cmd+Q on EasyPlaintextQuicklookExtension); the extension keeps working without it.

## 8. (Optional) Commit the change

If the working directory is a git repo and the user wants the change committed:

```sh
git add PreviewExtension/Info.plist
git commit -m "Add <UTI> support to QLSupportedContentTypes"
```
