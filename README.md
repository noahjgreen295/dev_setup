# dev_setup

Noah's basic dev setup, with zsh, GitHub over SSH, VS Code for Python, Claude Code and Docker. It covers two kinds of machine:

- **Windows**, doing its development work in WSL (Ubuntu). This is the main setup.
- **Mac**, working directly in macOS.

The guides are written for WSL first. Where a Mac does something differently, the guide either has an **On a Mac** section near the top or a separate Mac guide replaces it (listed below). Plain Linux should follow the WSL path, skipping the WSL-specific steps.

## Order

Work through the HOWTOs in this order. Later ones assume the earlier ones are done.

1. [HOWTO_wsl_setup.md](howtos/HOWTO_wsl_setup.md): install WSL and Ubuntu.
   - **Mac:** [HOWTO_mac_setup.md](howtos/HOWTO_mac_setup.md) instead: command line tools, Homebrew, Python.
2. [HOWTO_shell_config.md](howtos/HOWTO_shell_config.md): zsh, Oh My Zsh, aliases and theme.
3. [HOWTO_basic_github_setup.md](howtos/HOWTO_basic_github_setup.md): git identity and SSH keys so `git` can push and pull from GitHub.
4. [HOWTO_vs_code_install.md](howtos/HOWTO_vs_code_install.md): install VS Code, connect it to WSL, add Python and other extensions.
   - **Mac:** [HOWTO_vs_code_install_mac.md](howtos/HOWTO_vs_code_install_mac.md) instead.
5. [HOWTO_vs_code_settings_sync.md](howtos/HOWTO_vs_code_settings_sync.md): sync VS Code settings and extensions across computers.
6. [HOWTO_claude_code_in_wsl_and_vs_code.md](howtos/HOWTO_claude_code_in_wsl_and_vs_code.md): Claude Code on the command line and in VS Code.
7. [HOWTO_claude_to_github.md](howtos/HOWTO_claude_to_github.md): give Claude access to GitHub.
8. [HOWTO_docker_install_wsl_vscode.md](howtos/HOWTO_docker_install_wsl_vscode.md): Docker Desktop, used from WSL. Not yet covered for Mac.

