# dev_setup

Noah's basic dev setup: a Windows machine doing its development work in WSL (Ubuntu), with zsh, GitHub over SSH, VS Code for Python, Claude Code and Docker.

## Order

Work through the HOWTOs in this order. Later ones assume the earlier ones are done.

1. [HOWTO_wsl_setup.md](howtos/HOWTO_wsl_setup.md): install WSL and Ubuntu.
2. [HOWTO_shell_config.md](howtos/HOWTO_shell_config.md): zsh, Oh My Zsh, aliases and theme.
3. [HOWTO_basic_github_setup.md](howtos/HOWTO_basic_github_setup.md): SSH keys so `git` can push and pull from GitHub.
4. [HOWTO_vs_code_setup_including_wsl.md](howtos/HOWTO_vs_code_setup_including_wsl.md): install VS Code, connect it to WSL, add the Python extensions.
5. [HOWTO_vs_code_settings_sync.md](howtos/HOWTO_vs_code_settings_sync.md): sync VS Code settings and extensions across computers.
6. [HOWTO_claude_code_in_wsl_and_vs_code.md](howtos/HOWTO_claude_code_in_wsl_and_vs_code.md): Claude Code on the WSL command line and in VS Code.
7. [HOWTO_claude_to_github.md](howtos/HOWTO_claude_to_github.md): give Claude access to GitHub.
8. [HOWTO_docker_install_wsl_vscode.md](howtos/HOWTO_docker_install_wsl_vscode.md): Docker Desktop, used from WSL.

## Note on bash vs zsh

From step 2 on, the shell is zsh, which reads `~/.zshrc`, not `~/.bashrc`. Where a later HOWTO says to add a line to `~/.bashrc` (for example the `keychain` line in the GitHub HOWTO, or the `PATH` fix in the Claude Code HOWTO), put it in `~/.zshrc` instead.
