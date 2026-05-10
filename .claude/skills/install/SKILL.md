---
name: install
description: Use this skill when the user wants to install, build, or set up easy-plaintext-quicklook-extension on their Mac so that YAML and TOML files preview in Finder via spacebar. Builds the bundled Xcode project, copies the resulting .app into /Applications/, registers the embedded Quick Look extension with macOS, and verifies registration.
---

# /install — build and install easy-plaintext-quicklook-extension

Run this sequence in order. If a step fails, surface the error to the user and stop — do not silently retry.

## 0. Pre-flight

Confirm the working directory contains `EasyPlaintextQuicklookExtension.xcodeproj/`. If it does not, ask the user where they cloned the repo and `cd` there.

## 1. Confirm signing is set up

```sh
xcodebuild -showBuildSettings -project EasyPlaintextQuicklookExtension.xcodeproj \
  -target EasyPlaintextQuicklookExtension -configuration Debug 2>/dev/null \
  | grep -E '^\s*DEVELOPMENT_TEAM ='
```

If `DEVELOPMENT_TEAM` is empty or shows `AMXKCKJQUB` (the original author's team — only Justin Pearson can build with that), the user must set their own team before we can build. Tell them:

> Open `EasyPlaintextQuicklookExtension.xcodeproj` in Xcode, click the project root in the navigator, and for each of the two targets (`EasyPlaintextQuicklookExtension` and `PreviewExtension`) open the **Signing & Capabilities** tab and select **your** Apple Developer team in the **Team** dropdown. Xcode rewrites `project.pbxproj` automatically. Save (⌘S) and close Xcode, then ask me to re-run /install.

If the user is the original author (their team ID is `AMXKCKJQUB`), proceed.

## 2. Build

```sh
xcodebuild -project EasyPlaintextQuicklookExtension.xcodeproj \
           -scheme EasyPlaintextQuicklookExtension \
           -configuration Debug build 2>&1 | tail -3
```

Look for `** BUILD SUCCEEDED **`. On failure, show the relevant error lines and stop.

## 3. Locate the built bundle

```sh
xcodebuild -showBuildSettings -project EasyPlaintextQuicklookExtension.xcodeproj \
  -scheme EasyPlaintextQuicklookExtension -configuration Debug 2>/dev/null \
  | grep -E '^\s*BUILT_PRODUCTS_DIR =' | awk -F' = ' '{print $2}'
```

The built `.app` is at `<BUILT_PRODUCTS_DIR>/EasyPlaintextQuicklookExtension.app`.

## 4. Install into /Applications

```sh
SRC=<BUILT_PRODUCTS_DIR>/EasyPlaintextQuicklookExtension.app
DEST=/Applications/EasyPlaintextQuicklookExtension.app
[ -e "$DEST" ] && rm -rf "$DEST"
cp -R "$SRC" "$DEST"
open "$DEST"
sleep 2
```

The `open` triggers macOS LaunchServices to scan the bundle and register the embedded extension with `pluginkit`.

## 5. Reset the Quick Look daemon

```sh
killall QuickLookUIService 2>&1 || true
```

## 6. Verify registration

```sh
pluginkit -m -i com.justinppearson.EasyPlaintextQuicklookExtension.PreviewExtension
```

Output should be one line with a leading `+`, meaning enabled. If it shows `-` (disabled):

```sh
pluginkit -e use -i com.justinppearson.EasyPlaintextQuicklookExtension.PreviewExtension
killall QuickLookUIService
```

If no line appears at all, registration silently failed. Tell the user to manually launch `/Applications/EasyPlaintextQuicklookExtension.app` once and re-run this skill.

## 7. Tell the user to verify

> Install complete. In Finder, navigate to the repo's `examples/` folder, click `yamllint.yml` (or `sample.toml`), and press spacebar. The file's text contents should appear in the Quick Look window.
>
> You can quit the host app window if it's still open (Cmd+Q). The extension keeps working without it.

If preview fails after this, point them at:

- **System Settings → General → Login Items & Extensions** → click the (i) next to **EasyPlaintextQuicklookExtension** → confirm the **Quick Look** toggle is on.
- Run `pluginkit -m -p com.apple.quicklook.preview | grep -i easy` to confirm exactly one entry exists.
- If a stale registration is interfering, run `pluginkit -r <path-to-old-appex>` to remove it, then re-install.
