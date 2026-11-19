# HOWTO: VS Code setup on a Mac

How to install VS Code on a Mac and add the extensions I use, mainly for Python. This is the Mac version of [HOWTO_vs_code_install.md](HOWTO_vs_code_install.md). Assumes [HOWTO_mac_setup.md](HOWTO_mac_setup.md) is done.

## How it differs from WSL

On a Mac there's no WSL layer. VS Code, the terminal, Python and your extensions all run on the Mac itself, so there's no VS Code Server, no "WSL: Ubuntu" indicator, and extensions install in one place. Most of the Windows guide's WSL material simply doesn't apply.

Shortcuts use **Cmd** where Windows uses **Ctrl**: **Cmd+Shift+P** for the Command Palette, **Cmd+Shift+X** for Extensions.

## Install

### 1. Install VS Code

Either:

- **Download it** from [code.visualstudio.com](https://code.visualstudio.com/). Pick **Apple silicon** or **Intel** to match your Mac, or **Universal**, which works on both. Open the `.dmg` and drag **Visual Studio Code.app** into **Applications**.
- **Or use Homebrew:**

    ```
    brew install --cask visual-studio-code
    ```

Open it from **Applications** (or Spotlight). To keep it in the Dock, Control-click its icon and choose **Options** > **Keep in Dock**.

VS Code updates itself; accept the prompt when it appears.

### 2. Put `code` on your PATH

This lets you open folders from Terminal with `code .`, the way you do from WSL.

1. In VS Code, press **Cmd+Shift+P** and run **Shell Command: Install 'code' command in PATH**. (Newer versions call it **Install 'code' and optionally 'copilot' commands in PATH** and then ask about GitHub Copilot CLI; choose **No** to add only `code`.)
2. Open a new Terminal window and try it:

    ```
    cd ~/projects/myproject
    code .
    ```

The `.` means "this folder", so the window opens on your project with the integrated terminal already in the right place.

## Extensions

Open Extensions (**Cmd+Shift+X**), search, click **Install**. That's all; there's no separate WSL install step.

### Python essentials

| Extension | ID | What it's for |
| --- | --- | --- |
| **Python** (Microsoft) | `ms-python.python` | Running, IntelliSense, picking an interpreter, testing. Installing it also brings in the next three. |
| ↳ **Pylance** | `ms-python.vscode-pylance` | Autocomplete, type checking, go-to-definition. |
| ↳ **Python Debugger** | `ms-python.debugpy` | Breakpoints and step-through debugging. |
| ↳ **Python Environments** | `ms-python.vscode-python-envs` | Creating and switching between venvs. |
| **Ruff** (Astral) | `charliermarsh.ruff` | Linting and formatting. **Install this.** |
| **Even Better TOML** (tamasfe) | `tamasfe.even-better-toml` | `pyproject.toml` support. **Install this too.** |

Only if you use notebooks: **Jupyter** (`ms-toolsai.jupyter`).

### Ruff: strongly recommended

Ruff replaces flake8, Black, isort and a pile of pylint checks with one tool, and it's fast enough that it runs on every keystroke without you noticing. It flags unused imports, undefined names and likely bugs as you type, and fixes most of them itself on save. One extension, nothing else to configure, and every project ends up formatted the same way.

Open **Cmd+Shift+P** > **Preferences: Open User Settings (JSON)** and add:

```json
"[python]": {
  "editor.defaultFormatter": "charliermarsh.ruff",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll": "explicit",
    "source.organizeImports": "explicit"
  }
}
```

Every save then formats the file, applies Ruff's safe fixes and sorts the imports.

Project-specific Ruff rules (line length, which checks to turn on) go in a `[tool.ruff]` section of the project's `pyproject.toml`. The extension picks them up automatically.

### Even Better TOML: strongly recommended

Modern Python projects keep everything in `pyproject.toml`: dependencies, build settings, and config for Ruff, pytest and mypy. Without this extension VS Code treats it as plain text. With it you get colour, error checking as you type, formatting, and autocomplete for the known sections (it knows what belongs under `[project]` and `[tool.ruff]`, for example), so typos get caught before they turn into a confusing build error.

### Other extensions I use

There are even more, but they are covered in separate HOWTO docs. For now just these additional ones:

| Extension | Publisher | What it's for |
| --- | --- | --- |
| **Rainglow** | Dayle Rees (`daylerees.rainglow`) | 320+ colour themes, including Codecourse Contrast (below). |
| **Snowflake Theme** | breberaf | Colour theme in Snowflake's colours. |
| **IBM Color Scheme [DARK]** | PaperFanz | Dark colour theme based on IBM's palette. |
| **Material Icon Theme** | Philipp Kief (`PKief.material-icon-theme`) | File and folder icons in the Explorer, by file type. |

### Pick the Codecourse Contrast theme

1. Install **Rainglow**.
2. Open **Cmd+Shift+P** > **Preferences: Color Theme** (or **Cmd+K** then **Cmd+T**).
3. Type `codecourse`. Three show up: **Codecourse (rainglow)**, **Codecourse Contrast (rainglow)** and **Codecourse Light (rainglow)**.
4. Arrow through them to preview, and press **Enter** on **Codecourse Contrast (rainglow)**.

That sets this in your user settings:

```json
"workbench.colorTheme": "Codecourse Contrast (rainglow)"
```

For the file icons, run **Cmd+Shift+P** > **Material Icons: Activate Icon Theme**.

### Point VS Code at your venv

With a `.venv` in the project (`pvv` from [HOWTO_shell_config.md](HOWTO_shell_config.md)), run **Cmd+Shift+P** > **Python: Select Interpreter** and pick the one in `./.venv/bin/python`. VS Code usually spots it on its own. New integrated terminals then start with the venv already active.

### Hide cache and venv folders from the Explorer

Python, pytest and Ruff all leave cache folders behind (`__pycache__`, `.pytest_cache`, `.ruff_cache`), and every project has a `.venv`. They clutter the Explorer and you rarely need to open them. Open **Cmd+Shift+P** > **Preferences: Open User Settings (JSON)** and add:

```json
"files.exclude": {
  "**/__pycache__": true,
  "**/.pytest_cache": true,
  "**/.ruff_cache": true,
  "**/.venv": true
}
```

The `**/` prefix matches at any depth. This only hides the folders in VS Code (Explorer and search); they're still on disk, and VS Code still finds and uses the `.venv` interpreter.

Alternatively, turn on `explorer.excludeGitIgnore` to hide everything your `.gitignore` lists. A standard Python `.gitignore` already lists all of the above.

## Settings Sync

If you already use [Settings Sync](HOWTO_vs_code_settings_sync.md) on Windows, turn it on here with the same GitHub account and most of the above arrives on its own: settings, themes and extensions. Unlike WSL, there's no extra step to copy extensions across. Keyboard shortcuts are kept separately per OS by default, so the Mac's Cmd shortcuts and Windows' Ctrl ones don't overwrite each other.

## Troubleshooting

- **`code: command not found`.** Run the Shell Command from step 2 again, then open a new Terminal window. If an old `code` alias is left in `~/.zshrc` or `~/.bash_profile` from an earlier setup, remove it first.
- **VS Code won't update, or the `code` command breaks after a restart.** Make sure the app is in **Applications**, not still in **Downloads**. Move it there and rerun the Shell Command.
- **Python can't find packages you installed.** VS Code is using a different interpreter. Run **Python: Select Interpreter** and choose `.venv/bin/python`.
- **macOS asks whether VS Code may access Desktop, Documents or other folders.** That's a macOS privacy prompt. Allow it for folders you keep code in.

## Sources

- [Visual Studio Code on macOS (VS Code docs)](https://code.visualstudio.com/docs/setup/mac)
- [Getting started with Python in VS Code (VS Code docs)](https://code.visualstudio.com/docs/python/python-tutorial)
- [Formatting Python in VS Code (VS Code docs)](https://code.visualstudio.com/docs/python/formatting)
- [User interface: hiding files and folders (VS Code docs)](https://code.visualstudio.com/docs/getstarted/userinterface)
- [Settings Sync (VS Code docs)](https://code.visualstudio.com/docs/configure/settings-sync)
- [Python extension (VS Code Marketplace)](https://marketplace.visualstudio.com/items?itemName=ms-python.python)
- [Ruff extension (VS Code Marketplace)](https://marketplace.visualstudio.com/items?itemName=charliermarsh.ruff)
- [Even Better TOML (VS Code Marketplace)](https://marketplace.visualstudio.com/items?itemName=tamasfe.even-better-toml)
- [Rainglow (VS Code Marketplace)](https://marketplace.visualstudio.com/items?itemName=daylerees.rainglow)
- [Material Icon Theme (VS Code Marketplace)](https://marketplace.visualstudio.com/items?itemName=PKief.material-icon-theme)
