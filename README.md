# TRLauncher releases

Serves the update manifest the launcher checks on startup.

Kept separate from the gallery repo on purpose: the Discord bot's token can
write to the gallery, and a credential used for posting images must not also be
able to publish an executable that runs as administrator.

## Publishing an update

1. Bump `version` in `src-tauri/Cargo.toml` and `src-tauri/tauri.conf.json`
2. `npm run build`
3. `md5sum src-tauri/target/release/GTRlauncher.exe`
4. Copy the exe to `beta/GTRlauncher.exe` in this repo
5. Update `launcher.json` and `beta/launcher.json` here (`version`, `md5`, `notes`) — **always fill in `md5`**
6. Commit and push

The hash is the only thing verifying the download before it replaces a
launcher that runs with admin rights. An empty `md5` skips that check.
