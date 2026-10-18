# HOWTO: VS Code setup, including WSL

How to install VS Code on Windows, use it on projects inside WSL, and add the extensions needed for Python. Assumes [HOWTO_wsl_setup.md](HOWTO_wsl_setup.md) is done.

## How VS Code and WSL fit together

You install VS Code **once, on Windows**. You don't install a Linux copy of VS Code inside WSL.

When you open a WSL folder, VS Code puts a small helper, the **VS Code Server**, inside WSL. The window you see is Windows VS Code. The terminal, Python, the debugger and most extensions run inside Linux through the server. That's what makes Linux paths, the Linux `python3` and your `.venv` work as if VS Code were a Linux app.

So "installing VS Code in WSL" really means two things: install the **WSL extension** on the Windows side, then run `code .` from WSL once. That sets up the server.

## Install on Windows

### 1. Install VS Code

Download the Windows **User Installer** from [code.visualstudio.com](https://code.visualstudio.com/) and run it. Or, in PowerShell:

```
winget install -e --id Microsoft.VisualStudioCode
```

On the installer's **Select Additional Tasks** page:

- Keep **Add to PATH** ticked. This is what lets you type `code` in WSL. Without it, nothing in the next section works.
- Ticking the two **Add "Open with Code" action** boxes is handy too: right-click a folder in File Explorer to open it.

### 2. Install the WSL extension

Open VS Code, go to Extensions (**Ctrl+Shift+X**), search for **WSL**, and install the one from Microsoft (`ms-vscode-remote.remote-wsl`).

## Set up WSL

### 3. Run `code .` from WSL

Open your WSL (Ubuntu) terminal, go to a project folder, and run:

```
cd ~/projects/myproject
code .
```

The first time, it spends a moment installing the VS Code Server into WSL. After that it opens straight away.

Check the bottom-left corner of the window: it should say **WSL: Ubuntu**. That's how you know the window is running inside WSL.

### Why `code .` matters

`code .` is the main way to open WSL projects, and it's worth making a habit:

- **It opens the right kind of window.** Started from WSL, the window is connected to WSL automatically. If you instead open VS Code from the Start menu and use **File > Open Folder** on `\\wsl$\...`, you get a plain Windows window looking at Linux files over a slow network share. Python, the terminal and extensions are then the Windows ones, and things quietly misbehave.
- **It installs and updates the server.** The first `code .` sets up WSL support, and later runs keep the server in step with Windows VS Code after updates.
- **It opens the folder you're in.** The `.` means "this folder", so you start in the project with the terminal already in the right place.

If the corner doesn't say **WSL: Ubuntu**, close the window and reopen with `code .` from the WSL terminal.

Other ways in, once it's set up:

- From VS Code: click the remote button in the bottom-left corner > **Connect to WSL**, then **File > Open Folder** and pick a Linux path like `/home/<you>/projects/myproject`.
- **File > Open Recent** remembers WSL folders and reopens them in WSL.
- To jump back to Windows, run **WSL: Reopen in Windows** from the Command Palette, or use the remote button > **Close Remote Connection**.

Keep projects under `~/projects` in Linux, not under `/mnt/c/...`. Files on the Windows side are much slower to work with from WSL.

## Extensions for Python

### The essentials

| Extension | ID | What it's for |
| --- | --- | --- |
| **WSL** (Microsoft) | `ms-vscode-remote.remote-wsl` | Connects VS Code to WSL. Install on Windows (step 2). |
| **Python** (Microsoft) | `ms-python.python` | Running, IntelliSense, picking an interpreter, testing. Installing it also brings in the next three. |
| ↳ **Pylance** | `ms-python.vscode-pylance` | Autocomplete, type checking, go-to-definition. |
| ↳ **Python Debugger** | `ms-python.debugpy` | Breakpoints and step-through debugging. |
| ↳ **Python Environments** | `ms-python.vscode-python-envs` | Creating and switching between venvs. |
| **Ruff** (Astral) | `charliermarsh.ruff` | Linting and formatting in one fast tool. Replaces flake8, Black and isort. |

Only if you use notebooks: **Jupyter** (`ms-toolsai.jupyter`).

### Set Ruff as the formatter

Open **Ctrl+Shift+P** > **Preferences: Open User Settings (JSON)** and add:

```json
"[python]": {
  "editor.defaultFormatter": "charliermarsh.ruff",
  "editor.formatOnSave": true
}
```

Files then tidy themselves each time you save.

### Point VS Code at your venv

With a `.venv` in the project (`pvv` from [HOWTO_shell_config.md](HOWTO_shell_config.md)), run **Ctrl+Shift+P** > **Python: Select Interpreter** and pick the one in `./.venv/bin/python`. VS Code usually spots it on its own. New integrated terminals then start with the venv already active.

## Installing extensions in Windows and in WSL

Extensions come in two kinds, and VS Code decides where each goes:

- **UI extensions** (themes, icon packs, keymaps) run on Windows and work in every window, WSL or not.
- **Workspace extensions** (Python, Pylance, Ruff, debuggers, Claude Code) need to run where the code is. For WSL windows, that means installing them **inside WSL** separately. A Windows install doesn't count.

### Windows (local) windows

Open Extensions (**Ctrl+Shift+X**), search, click **Install**. That's it.

### WSL windows

Open a WSL window first (`code .`), then open Extensions. You'll see two groups:

- **Local - Installed:** what's on Windows. Workspace extensions here show greyed out with a note that they need installing in WSL.
- **WSL: Ubuntu - Installed:** what's in WSL.

To install:

- **One extension:** search for it and click **Install in WSL: Ubuntu**. Or, for a greyed-out one under **Local - Installed**, click its **Install in WSL: Ubuntu** button.
- **Everything at once:** hover over the **Local - Installed** heading and click the cloud icon (**Install Local Extensions in 'WSL: Ubuntu'...**). Tick the ones you want and click **OK**.

The recommended order on a new machine: install the Python extensions on Windows first, open a WSL window with `code .`, then use the cloud icon to copy them all across.

Every WSL distro gets its own set, so if you add a second distro, do this again there.

### Settings Sync and WSL

[Settings Sync](HOWTO_vs_code_settings_sync.md) carries settings and Windows-side extensions to a new computer, but it does **not** install extensions inside WSL. On a new machine, after sync finishes, use the cloud icon above to copy them into WSL.

## Troubleshooting

- **`code: command not found` in WSL.** Close and reopen the WSL terminal. If it's still missing, reinstall VS Code with **Add to PATH** ticked, then run `wsl --shutdown` in PowerShell and reopen Ubuntu.
- **The bottom-left corner doesn't say WSL.** You're in a Windows window. Reopen with `code .` from WSL, or run **WSL: Reopen Folder in WSL** from the Command Palette.
- **An extension seems installed but does nothing in WSL.** It's only installed on Windows. Check the **WSL: Ubuntu - Installed** group and install it there.
- **Python can't find packages you installed.** VS Code is using a different interpreter. Run **Python: Select Interpreter** and choose `.venv/bin/python`.
- **The WSL window won't connect.** Run **WSL: Show Log** from the Command Palette. Running `wsl --shutdown` in PowerShell and trying again fixes most cases.

## Sources

- [Developing in WSL (VS Code docs)](https://code.visualstudio.com/docs/remote/wsl)
- [Getting started with Python in VS Code (VS Code docs)](https://code.visualstudio.com/docs/python/python-tutorial)
- [Formatting Python in VS Code (VS Code docs)](https://code.visualstudio.com/docs/python/formatting)
- [Python extension (VS Code Marketplace)](https://marketplace.visualstudio.com/items?itemName=ms-python.python)
