# HOWTO: Mac setup

How to get a Mac ready for the other HOWTOs in this repo: Apple's command line tools, Homebrew, Python and a projects folder. This is the Mac counterpart to [HOWTO_wsl_setup.md](HOWTO_wsl_setup.md). On a Mac there's no Linux layer to install: the Terminal app already gives you a Unix shell, and zsh is the default.

## What you need

- A recent version of macOS. Claude Code needs macOS 13 (Ventura) or later.
- An admin account on the Mac. Homebrew and the command line tools ask for its password.

Find your macOS version under Apple menu > **About This Mac**. Check whether the Mac has an Apple silicon (M-series) or Intel chip while you're there; a few steps differ.

## Install

### 1. Open Terminal

Press **Cmd+Space**, type `Terminal`, and press Enter. Everything below runs there.

### 2. Install the Xcode Command Line Tools

These give you `git`, `make`, a C compiler and other tools that Homebrew and Python packages need. You don't need the full Xcode app.

```
xcode-select --install
```

A dialog opens. Click **Install** and wait for it to finish. Then check:

```
git --version
```

If it says the tools are already installed, you're done with this step.

### 3. Install Homebrew

Homebrew is the Mac's package manager, playing the part `apt` plays on Ubuntu. Where another HOWTO says `sudo apt install <thing>`, on a Mac it's usually `brew install <thing>`, without `sudo`.

```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

The script says what it will do and pauses before doing it. Press Enter to continue and enter your Mac password when asked.

When it finishes, read the **Next steps** it prints. On an Apple silicon Mac, Homebrew lives in `/opt/homebrew`, which isn't on your PATH yet, and the script gives you the commands to fix that. They look like this; run the ones it prints:

```
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```

On an Intel Mac, Homebrew uses `/usr/local`, which is already on the PATH.

Check it:

```
brew --version
brew doctor
```

`brew doctor` should say `Your system is ready to brew.`

### 4. Install Python

macOS comes with a `python3`, but it's the one bundled with the command line tools: an older version meant for Apple's own use. Install a current one with Homebrew:

```
brew install python
```

Open a new terminal and check:

```
which python3
python3 --version
```

`which` should print `/opt/homebrew/bin/python3` (Apple silicon) or `/usr/local/bin/python3` (Intel). The venv module is included, so unlike Ubuntu there's no separate `python3-venv` package.

Homebrew's Python refuses `pip install` outside a virtual environment. That's on purpose; make a venv per project (the `pvv` and `sva` aliases in [HOWTO_shell_config.md](HOWTO_shell_config.md)).

### 5. Make a projects folder

```
mkdir ~/projects
```

Keep it directly under your home folder, not in **Desktop** or **Documents**. If iCloud's **Desktop & Documents Folders** option is on, anything in those folders is synced to iCloud, and syncing a repo's `.git` folder and `.venv` causes slowdowns and conflicts.

To open a folder in Finder from Terminal, run `open .`.

## Keep things updated

- **Homebrew packages:** `brew update && brew upgrade` every so often.
- **macOS and the command line tools:** **System Settings** > **General** > **Software Update**.

## Useful commands

| Command | What it does |
| --- | --- |
| `brew install <name>` | Install a command-line tool |
| `brew install --cask <name>` | Install a Mac app (VS Code, Docker Desktop and so on) |
| `brew upgrade` | Upgrade everything Homebrew installed |
| `brew list` | List what Homebrew installed |
| `open .` | Open the current folder in Finder |
| `pbcopy < file` | Copy a file's contents to the clipboard |

## Troubleshooting

- **`brew: command not found` after installing.** The PATH step in step 3 wasn't done. Run the two `shellenv` lines, then open a new terminal.
- **`xcrun: error: invalid active developer path` when running `git`.** A macOS upgrade removed the command line tools. Run `xcode-select --install` again.
- **`python3` is still Apple's version.** Open a new terminal. If `which python3` still prints `/usr/bin/python3`, Homebrew isn't on your PATH; see the first item.
- **`error: externally-managed-environment` from `pip`.** You're installing outside a venv. Make and activate one first.

## Sources

- [Homebrew](https://brew.sh/)
- [Homebrew installation (Homebrew docs)](https://docs.brew.sh/Installation)
- [Advanced setup (Claude Code docs)](https://code.claude.com/docs/en/setup)
