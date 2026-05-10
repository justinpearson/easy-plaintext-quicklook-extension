---
name: uninstall
description: Use this skill when the user wants to uninstall, remove, or undo the easy-plaintext-quicklook-extension. Force-drops the cached pluginkit registration so macOS forgets the extension immediately, deletes the .app from /Applications, and resets the Quick Look daemon so YAML/TOML files revert to the generic-document icon.
---

# /uninstall — remove easy-plaintext-quicklook-extension

Run this sequence in order.

## 1. Pre-flight

```sh
ls /Applications/EasyPlaintextQuicklookExtension.app 2>/dev/null && echo "found" || echo "not found"
```

If the `.app` is not installed in `/Applications/`, also check `pluginkit` for any stale registration that may still be active:

```sh
pluginkit -m -p com.apple.quicklook.preview | grep -i easyplaintext
```

If neither the `.app` nor the registration is present, tell the user "already uninstalled" and stop.

## 2. Drop the pluginkit registration first

If a registration exists, remove it before deleting the `.app`. This avoids a window where the registration points to a deleted path.

```sh
APPEX=/Applications/EasyPlaintextQuicklookExtension.app/Contents/PlugIns/PreviewExtension.appex
[ -e "$APPEX" ] && pluginkit -r "$APPEX"
```

If there are stale registrations elsewhere (e.g. pointing into DerivedData from a developer build), surface them to the user and offer to remove them too:

```sh
pluginkit -m -i com.justinppearson.EasyPlaintextQuicklookExtension.PreviewExtension -vv
```

Each `Path =` line shows where macOS thinks an active registration lives. For each one, `pluginkit -r <path>` to remove it.

## 3. Delete the .app

Default to moving to Trash (so the user can recover if needed):

```sh
osascript -e 'tell application "Finder" to delete POSIX file "/Applications/EasyPlaintextQuicklookExtension.app"'
```

If the user prefers a hard delete:

```sh
rm -rf /Applications/EasyPlaintextQuicklookExtension.app
```

## 4. Reset the Quick Look daemon

```sh
killall QuickLookUIService 2>&1 || true
```

## 5. Verify

```sh
pluginkit -m -p com.apple.quicklook.preview | grep -i easyplaintext || echo "(no registrations remain)"
ls /Applications/EasyPlaintextQuicklookExtension.app 2>/dev/null || echo "(.app removed)"
```

Both lines should report absence.

## 6. Tell the user

> Uninstall complete. Spacebar a `.yaml` or `.toml` file in Finder; you should now see the generic-document icon, confirming the extension is no longer claiming those file types.
>
> If you also want to remove the System Settings entry, it should disappear automatically once macOS notices the bundle is gone (System Settings → General → Login Items & Extensions). If it persists, log out and back in.
