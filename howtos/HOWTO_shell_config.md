# HOWTO: Shell config (zsh + Oh My Zsh)

How to switch WSL (Ubuntu) from bash to zsh, add Oh My Zsh, and set up the aliases and theme I use. Assumes [HOWTO_wsl_setup.md](HOWTO_wsl_setup.md) is done.

## Install zsh

### 1. Install it

```
sudo apt update
sudo apt install zsh
```

### 2. Check the version

```
zsh --version
```

Oh My Zsh wants 5.0.8 or newer. Current Ubuntu ships well past that.

### 3. Make zsh the default shell

```
chsh -s $(which zsh)
```

It asks for your Linux password. Then close the terminal and open a new one (or run `wsl --shutdown` in PowerShell and reopen Ubuntu). Check it took:

```
echo $SHELL
```

It should print `/usr/bin/zsh` or `/bin/zsh`.

The first time zsh starts with no `~/.zshrc`, it shows a setup menu. Press `q` to skip it; step 4 writes a `~/.zshrc` for you.

## Install Oh My Zsh

### 4. Run the installer

Oh My Zsh needs `git` and `curl`, which Ubuntu already has (`sudo apt install git curl` if not).

```
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

- It installs into `~/.oh-my-zsh`.
- It writes a new `~/.zshrc`. If you already had one, it's renamed to `~/.zshrc.pre-oh-my-zsh`; copy anything you want to keep from there.
- If you skipped step 3, it offers to make zsh your default shell. Say yes.

Update it later with `omz update`.

## Configure

All of this goes in `~/.zshrc`. After editing, run `source ~/.zshrc` or open a new terminal.

### 5. Aliases

Add these at the **end** of `~/.zshrc`, below the `source $ZSH/oh-my-zsh.sh` line, so they win over any Oh My Zsh alias with the same name:

```
alias gsun="git status -uno"
alias ls="ls -F"
alias sva="source .venv/bin/activate"
alias pvv="python3 -m venv .venv"
```

| Alias | Runs | What it's for |
| --- | --- | --- |
| `gsun` | `git status -uno` | Git status without untracked files (`-uno` = `--untracked-files=no`), so only changes to tracked files show. |
| `ls` | `ls -F` | Marks entries by type: `/` for directories, `*` for executables, `@` for symlinks. |
| `sva` | `source .venv/bin/activate` | Activate the Python virtual environment in the current folder. Leave it with `deactivate`. |
| `pvv` | `python3 -m venv .venv` | Make a new virtual environment in `.venv` in the current folder. |

A typical new Python project: `pvv`, then `sva`, then `pip install ...`.

`pvv` needs the venv module. On Ubuntu that's a separate package:

```
sudo apt install python3-venv
```

Oh My Zsh normally aliases `ls` to `ls --color=tty`. Because the `ls` alias above comes after it, it replaces that one and listings lose their color. To keep both:

```
alias ls="ls -F --color=tty"
```

See every alias in effect with `alias`, or one with `alias gsun`.

### 6. Theme

My `~/.zshrc` sets:

```
ZSH_THEME="midnight"
```

**Note: we can't find where this theme came from.** `midnight` is not one of the themes bundled with Oh My Zsh, and it isn't on the [external themes list](https://github.com/ohmyzsh/ohmyzsh/wiki/External-themes) or anywhere obvious on GitHub. It's a custom file sitting in `~/.oh-my-zsh/custom/themes/` (or `~/.oh-my-zsh/custom/`) on the existing WSL install. Oh My Zsh looks in those folders before its own `themes/` folder.

On a new machine, until the file is tracked down:

- **Copy it from the old machine** if you still have it: put `midnight.zsh-theme` in `~/.oh-my-zsh/custom/themes/` on the new one.
- **Otherwise pick a bundled theme**, or leave the line as is and you'll see `[oh-my-zsh] theme 'midnight' not found` with zsh's plain prompt. Browse the bundled ones on the [Themes wiki page](https://github.com/ohmyzsh/ohmyzsh/wiki/Themes); `robbyrussell` is the default.

TODO: commit `midnight.zsh-theme` to this repo so it can be copied in directly.

## Troubleshooting

- **`chsh: PAM: Authentication failure`.** Wrong Linux password. If you've forgotten it, see the WSL HOWTO.
- **Still in bash after `chsh`.** WSL keeps the old session alive. Run `wsl --shutdown` in PowerShell, then reopen Ubuntu.
- **Aliases or theme not working.** Check the lines are in `~/.zshrc` (not `~/.bashrc`), then `source ~/.zshrc`.
- **Things you set up in `~/.bashrc` before switching are gone.** zsh doesn't read `~/.bashrc`. Copy those lines into `~/.zshrc`.

## Sources

- [Installing ZSH (Oh My Zsh wiki)](https://github.com/ohmyzsh/ohmyzsh/wiki/Installing-ZSH)
- [Oh My Zsh README](https://github.com/ohmyzsh/ohmyzsh)
- [Themes (Oh My Zsh wiki)](https://github.com/ohmyzsh/ohmyzsh/wiki/Themes)
