# HOWTO: VS Code Settings Sync

How to turn on VS Code's built-in Settings Sync, so settings, keyboard shortcuts and extensions follow you to every computer where you sign in.

## Do I need SSH keys for this?

No. Settings Sync doesn't use git or SSH. You sign in to a GitHub (or Microsoft) account in the browser, and VS Code stores your settings in Microsoft's sync service. It doesn't go into a repo.

The SSH setup in [HOWTO_basic_github_setup.md](HOWTO_basic_github_setup.md) is for `git push` and `git pull` from the command line. You'll want it on any machine you code on, but Settings Sync works with or without it.

## What syncs

- Settings (user settings, not workspace ones)
- Keyboard shortcuts (kept separately per OS by default)
- Snippets
- User tasks
- UI state (layout, display language, "don't show again" choices)
- Extensions, and whether each is enabled
- Profiles

What **doesn't** sync:

- Settings marked machine-specific (paths and the like)
- Workspace tasks and settings. Those live in the project's `.vscode/` folder, so commit them with the project if you want them everywhere.
- **Extensions installed in WSL.** See the WSL section below.

## Turn it on

### 1. Start sync

Click the gear icon (bottom left) or the person icon above it, and choose **Backup and Sync Settings...**

### 2. Pick what to sync

Untick anything you want to keep separate per machine. Leaving everything on is fine.

### 3. Sign in

Click **Sign in** and choose **GitHub**. A browser window opens. Approve the request and you're sent back to VS Code. It's the same GitHub account you use for everything else.

### 4. First sync

Your local settings and whatever is in the cloud get merged, and from then on it syncs in the background.

If they can't be merged, VS Code asks what to do:

- **Accept Local / Replace Remote:** keep this machine's version and overwrite the cloud copy.
- **Accept Remote / Replace Local:** take the cloud version and overwrite this machine's.
- **Show Conflicts:** compare them side by side, edit, then click **Complete Merge**.

On a brand-new machine with nothing set up yet, **Replace Local** is usually what you want.

### 5. On every other computer

Do steps 1 to 4 with the same GitHub account.

## WSL

VS Code runs on Windows, so you sign in to Settings Sync on the Windows side, once. When you open a WSL folder (`code .` from WSL, or the **WSL** remote button), your synced settings and keybindings apply there too.

Extensions are the catch. Many extensions have to be installed inside WSL to work there, and Settings Sync doesn't sync extensions into or out of WSL. After opening a WSL window:

1. Open the Extensions view (**Ctrl+Shift+X**).
2. In the **Local - Installed** section, click the cloud icon (**Install Local Extensions in 'WSL: Ubuntu'...**).
3. Tick the ones you want and click **OK**.

Do this again when you add a new extension you also want in WSL.

## Leave something out

- **A setting:** in the Settings editor, click the gear next to the setting and choose **Don't Sync This Setting**. It's added to `settingsSync.ignoredSettings`.
- **An extension:** in the Extensions view, click its gear and untick **Sync This Extension**. It's added to `settingsSync.ignoredExtensions`.

## Useful commands

Open the Command Palette (**Ctrl+Shift+P**) and type:

| Command | What it does |
| --- | --- |
| `Settings Sync: Show Synced Data` | See what's in the cloud, earlier versions, and which machines are syncing |
| `Settings Sync: Sync Now` | Sync right away instead of waiting |
| `Settings Sync: Configure...` | Change which kinds of data sync |
| `Settings Sync: Open Local Backups Folder` | Local backups, kept for 30 days |
| `Settings Sync: Turn Off` | Stop syncing on this machine |

The cloud keeps your last 20 versions of each kind of data, so a bad change can be rolled back from **Show Synced Data**.

## Switching accounts or turning it off

- **Different account:** run `Settings Sync: Turn Off`, then turn it on again and sign in with the other account.
- **Off on this machine:** gear icon > **Settings Sync is On** > **Turn Off**.
- **Off everywhere and wipe the cloud copy:** same as above, but also tick **Turn off sync on all your devices and clear the data from the cloud**.

## Troubleshooting

- **Sync turned itself off, or a category stopped syncing.** Settings, keybindings and user tasks are each capped at 100 KB. Trim the file, then turn the category back on with `Settings Sync: Configure...`.
- **Sign-in doesn't stick, or sync keeps asking you to log in.** Check the **Account** and **Log (Settings Sync)** output channels (View > Output, then pick from the dropdown).
- **Insiders and regular VS Code don't share settings.** They use separate sync services by default. When turning sync on in Insiders, choose the regular (Stable) service.

## Not the old extension

There used to be a third-party extension also called "Settings Sync" (by Shan Khan) that stored everything in a GitHub Gist. It's unrelated and no longer needed. If it's installed, uninstall it before turning on the built-in one.

## Sources

- [Settings Sync (VS Code docs)](https://code.visualstudio.com/docs/configure/settings-sync)
