# HOWTO: WSL setup

How to install WSL (Windows Subsystem for Linux) with Ubuntu on Windows and get it ready to use. The other HOWTOs in this repo assume this is done.

## What you need

- Windows 11, or Windows 10 version 2004 (build 19041) or later. Check with **Win + R** > `winver`.
- Admin rights on the computer.

## Install

### 1. Run the installer

Open PowerShell **as administrator** (right-click Start > **Terminal (Admin)**) and run:

```
wsl --install
```

This turns on the Windows features WSL needs and installs Ubuntu as WSL 2. It only works when WSL isn't installed at all.

To install a different distro, list the options and pick one:

```
wsl --list --online
wsl --install -d <DistroName>
```

If the install sits at 0.0%, run `wsl --install --web-download -d Ubuntu` instead.

### 2. Restart the computer

The install isn't finished until you restart.

### 3. Create your Linux user

After the restart, Ubuntu opens a terminal on its own (or open **Ubuntu** from the Start menu). The first launch takes a minute while files unpack. When prompted, enter a username and password.

- They don't have to match your Windows login.
- The password shows nothing as you type. That's normal.
- You'll need it for `sudo`, so don't lose it.

### 4. Check you're on WSL 2

In PowerShell:

```
wsl -l -v
```

The VERSION column should say 2. If it says 1:

```
wsl --set-version Ubuntu 2
wsl --set-default-version 2
```

## Setup

### 5. Update Ubuntu's packages

In the Ubuntu terminal:

```
sudo apt update && sudo apt upgrade -y
```

Do this now and every so often after.

### 6. Update WSL itself

In PowerShell:

```
wsl --update
```

### 7. Make a projects folder

Keep your work in the Linux file system, not under `/mnt/c/...`. Crossing between Windows and Linux files is much slower.

```
mkdir ~/projects
```

To see Linux files from Windows, open File Explorer and go to `\\wsl$\Ubuntu\home\<username>`, or run `explorer.exe .` from any folder in WSL.

## Useful commands

All run in PowerShell.

| Command | What it does |
| --- | --- |
| `wsl` | Open the default distro in the current window |
| `wsl -l -v` | List installed distros and their WSL version |
| `wsl --shutdown` | Stop all running distros (fixes most oddities) |
| `wsl --set-default <DistroName>` | Change which distro `wsl` opens |
| `wsl --update` | Update WSL |

## Troubleshooting

- **`wsl --install` just prints help text.** WSL is already partly installed. Run `wsl --list --online`, then `wsl --install -d Ubuntu`.
- **Error mentioning virtualization or `0x80370102`.** Virtualization is off in the computer's BIOS/UEFI. Turn it on (often called Intel VT-x or AMD-V/SVM), then try again.
- **Forgot the Linux password.** In PowerShell run `wsl -u root`, then `passwd <username>`.

## Sources

- [Install WSL (Microsoft docs)](https://learn.microsoft.com/en-us/windows/wsl/install)
- [Set up a WSL development environment (Microsoft docs)](https://learn.microsoft.com/en-us/windows/wsl/setup/environment)
- [Basic commands for WSL (Microsoft docs)](https://learn.microsoft.com/en-us/windows/wsl/basic-commands)
