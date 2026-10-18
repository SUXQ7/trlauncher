# TRLauncher releases

Serves the update manifest the launcher checks on startup.

Kept separate from the gallery repo on purpose: the Discord bot's token can
write to the gallery, and a credential used for posting images must not also be
able to publish an executable that runs as administrator.

**Push access here is the ability to run code as administrator on every
player's machine.** The launcher downloads whatever `launcher.json` points at
and runs it elevated. Treat a collaborator on this repo the same way you would
treat a server root account.

---

## Before you release: check which servers the build points at

This is the first step because it is the one that cannot be undone quietly.
Every player gets the update automatically, and GitHub serves it for five
minutes before you could reverse it.

| file | constant | live value |
|---|---|---|
| `src-tauri/src/patcher.rs` | `CDN_BASES` | `https://cdn.talesrunner.us/TalesRunner/Patch/LastChaos/` |
| `src-tauri/src/main.rs` | `AUTH_URL` | `https://www.talesrunner.us/launcher/login.php` |

If either still points at a test box (`31.56.120.213`), publishing will send
every player's password to that box and start a full re-download of whatever
build it serves — tens of gigabytes, replacing their install. Check both.

---

## Publishing an update

1. **Bump the version in BOTH files.** `src-tauri/Cargo.toml` and
   `src-tauri/tauri.conf.json`. `Cargo.toml` is what the exe reports and what
   the updater compares; miss it and the new build still claims the old
   version, so nobody updates.

2. **Build.**
   ```
   npx tauri build
   ```

3. **Stage the exe in this repo.** The manifest points at the file on `main`,
   not at a GitHub release:
   ```
   copy ..\src-tauri\target\release\trlauncher.exe trlauncher.exe
   ```

4. **Read its hash.**
   ```
   (Get-FileHash trlauncher.exe -Algorithm MD5).Hash.ToLower()
   ```

5. **Update `launcher.json`** with the new version and that hash. `notes` is
   shown to players in the update prompt, so write it for them.
   ```json
   {
     "version": "1.4.5",
     "url": "https://raw.githubusercontent.com/SUXQ7/trlauncher/main/trlauncher.exe",
     "md5": "<the hash from step 4>",
     "notes": "What changed, in a sentence a player understands."
   }
   ```

6. **Commit and push** from this folder.
   ```
   git add -A && git commit -m "1.4.5 - what changed" && git push origin HEAD
   ```

7. **Verify what is actually served.** Not optional — this is the same check
   the updater runs, and a mismatch fails the update for every player at once.
   ```
   curl -s -o dl.exe https://raw.githubusercontent.com/SUXQ7/trlauncher/main/trlauncher.exe
   (Get-FileHash dl.exe -Algorithm MD5).Hash.ToLower()   # must equal the manifest md5
   ```

---

## What players see

At startup, before the window appears, the launcher fetches `launcher.json`
over HTTPS and compares versions numerically per component, so `1.3.10`
correctly beats `1.3.9`. If yours is newer it shows a Yes/No prompt with the
installed version, the available one and your `notes`.

**Yes** downloads with a progress window, checks the MD5, renames the running
exe to `.old`, moves the new build into place and relaunches. **No** starts the
launcher normally and asks again next time; there is no way to force an update.

Every failure path keeps the working launcher — unreachable GitHub, malformed
manifest, hash mismatch, interrupted download, a rename that fails. The worst
case of a bad release is that nobody updates, not that anybody is stuck.

---

## Three things that will catch you

**An empty `md5` skips verification entirely.** The check is skipped when the
field is blank, so a truncated or corrupted download would be renamed into
place and run with admin rights. Always fill it in.

**Bumping only one of the two version files.** See step 1.

**Shipping an exe whose version matches the published one.** Two different
binaries both calling themselves 1.4.5 means a player's log says one thing
while they are running another. Bump before handing a build to anyone, even a
tester.

---

## Propagation and rollback

GitHub raw caches for **300 seconds per edge node**, and the manifest and the
exe cache independently. For a few minutes some players see the new manifest
with the old binary still cached, or the reverse. Both resolve themselves.

To roll back, publish the previous binary under a **higher** version number.
Lowering the number does nothing: anyone already updated will not see a lower
version as newer.

The launcher binary is committed here on every release, so the repo grows by
roughly 20 MB each time.
