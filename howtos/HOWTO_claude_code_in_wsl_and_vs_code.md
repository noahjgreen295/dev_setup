# HOWTO: Set up Claude Code in WSL and VS Code

How to install and set up Claude Code on Windows with WSL, for two places: the WSL command line, and VS Code connected to WSL. Claude Code is not installed in PowerShell. Do Preliminaries first, then the section for each place you use.

For GitHub access, see [HOWTO_claude_to_github.md](HOWTO_claude_to_github.md).

## Preliminaries

### What you need

- A paid Claude plan (Pro, Max, Team or Enterprise) or a Claude Console account. The free plan doesn't include Claude Code.
- WSL 2. Claude Code fails on WSL 1 with `Exec format error`.

### Check you're on WSL 2

In PowerShell, run:

```
wsl -l -v
```

The VERSION column should say 2. If it says 1, convert the distro:

```
wsl --set-version <DistroName> 2
```

### How the two places relate

- **Separate installs.** The VS Code extension carries its own private copy of Claude Code for its chat panel. That copy doesn't make the `claude` command work in a terminal, so the command line needs its own install.
- **Shared settings and history.** Both read the same settings file, `~/.claude/settings.json` in WSL, and share conversation history. To continue a VS Code conversation in the terminal, run `claude --resume` and pick it from the list.
- **Shared MCP servers.** A server added in one place (with `/mcp` in the VS Code panel or `claude mcp add` in the terminal) shows up in the other.

### MCP servers and chat connectors

- **Chat connectors carry over to Claude Code.** MCP connectors you add in claude.ai chat (at claude.ai/customize/connectors), such as Google Calendar, appear in Claude Code automatically, in both the terminal and VS Code. In `/mcp` they're labeled as coming from claude.ai. This only works while Claude Code is signed in with your claude.ai account; run `/status` to check which login is active.
- **Claude Code servers don't carry over to chat.** A server added in Claude Code stays in Claude Code.
- **Duplicates.** If a server added in Claude Code points at the same URL as a chat connector, Claude Code uses its own copy and hides the connector.
- **Some connectors sign in only from chat.** Gmail, Google Calendar and Microsoft 365 can't be signed into from Claude Code. Add those in chat.
- **Hiding a connector in one project.** Toggle it off in `/mcp`. The change applies only to that project.
- **VS Code's own MCP gallery is separate.** Searching `@mcp` in VS Code's Extensions view installs servers for VS Code's own chat (Copilot), not for Claude Code.

### Keep projects in the Linux file system

Put projects under your WSL home, such as `~/projects`, not under `/mnt/c/...`. Working across the Windows and Linux boundary is noticeably slower.

## WSL command line

1. **Install.** Open a WSL terminal and run:

    ```
    curl -fsSL https://claude.ai/install.sh | bash
    ```

    The installer shows no progress while it downloads. Use this installer, not npm: npm inside WSL often picks up Windows' Node by mistake. This install updates itself automatically.

2. **Check it.** Open a new WSL terminal and run `claude --version`. It should print a version number.

3. **Sign in.** Run `claude` and follow the prompts. Because WSL is separate from your Windows browser, the browser often can't hand the login back to the terminal on its own. When it can't, it shows a code; paste it at the `Paste code here if prompted` prompt.

4. **Check the setup.** Run `claude doctor`. It checks the install and settings and suggests fixes for anything wrong.

### Optional: replace the spinner verbs

Claude Code rotates through whimsical verbs while it works. There's no setting to turn them off, but you can replace the whole list with one plain word. Add this to `~/.claude/settings.json`, merging it with anything already in the file, then restart `claude`:

```json
{
  "spinnerVerbs": {
    "mode": "replace",
    "verbs": ["Working"]
  }
}
```

`"replace"` uses only your verbs; `"append"` would add yours to the built-in ones. To also hide the "Cooked for 12s" line after each response, add `"showTurnDuration": false` to the same file.

### Troubleshooting

- **`claude: command not found`.** `~/.local/bin` isn't on your PATH. Run:

    ```
    echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
    source ~/.bashrc
    ```

- **`cannot execute binary file: Exec format error`.** You're on WSL 1. Convert to WSL 2; see Preliminaries.
- **The browser doesn't open at sign-in.** Press `c` at the login prompt to copy the URL, and open it in your Windows browser.
- **Pasting the login code does nothing.** Try right-click or Shift+Insert to paste, or run `claude auth login`, which reads the code from standard input.
- **`exec: node: not found`, or npm reports a platform mismatch.** You installed with npm and WSL is using Windows' Node. Uninstall it with `npm uninstall -g @anthropic-ai/claude-code` and use the installer in step 1.

## WSL VS Code

1. **Install the WSL extension.** In VS Code, install Microsoft's **WSL** extension if you don't already have it. You need VS Code 1.94.0 or later.
2. **Open a WSL window.** Click the remote button at the bottom left and connect to WSL, or run `code .` from a WSL terminal in your project folder.
3. **Install Claude Code in WSL.** In that WSL window, open Extensions (`Ctrl+Shift+X`), search for **Claude Code** from Anthropic, and install it. VS Code installs this kind of extension separately for each WSL distro, so the button may read **Install in WSL: Ubuntu**; that's the one you want. If it doesn't appear after installing, run Developer: Reload Window from the Command Palette.
4. **Open the panel.** Open any file and click the spark icon at the top right of the editor, or click **✻ Claude Code** in the status bar at the bottom right.
5. **Sign in.** Click **Sign in** and approve it in your browser.

### Use the command line inside VS Code

Open the integrated terminal with `` Ctrl+` ``. In a WSL window it's already a WSL terminal, so run `claude` there. It connects to VS Code on its own, so its proposed changes open in VS Code's side-by-side diff view. This needs the command-line install from the WSL command line section.

If you run `claude` in a terminal outside VS Code, type `/ide` to connect it.

### Useful shortcuts

- `Ctrl+Esc` switches focus between your code and the Claude panel.
- `Alt+K` inserts a reference to the current file and selected lines into your prompt.
- To open a new conversation in its own tab, run Claude Code: Open in New Tab from the Command Palette. Its listed shortcut, `Ctrl+Shift+Esc`, opens Task Manager on Windows instead.

### Optional settings

Open these with `Ctrl+Shift+P`, then Preferences: Open User Settings (JSON).

- **Where Claude opens.** `"sidebar"` opens it in the right sidebar; `"panel"`, the default, opens it as an editor tab. You can also drag the panel anywhere and the extension remembers where you put it.

    ```json
    "claudeCode.preferredLocation": "sidebar"
    ```

- **Replace the spinner verbs.** The chat panel has its own setting, separate from the command line's:

    ```json
    "claudeCode.spinnerVerbs": {
      "mode": "replace",
      "verbs": ["Working"]
    }
    ```

    Reload the window afterward.

### Troubleshooting

- **No spark icon.** Open a file; the icon doesn't appear with only a folder open. Or click **✻ Claude Code** in the status bar.
- **`claude` not found in the integrated terminal.** The extension doesn't add `claude` to your PATH. Install the command-line version (WSL command line section), then reopen the terminal.
- **Sign-in screen keeps coming back.** Run Developer: Reload Window from the Command Palette.

## Sources

- [Advanced setup (Claude Code docs)](https://code.claude.com/docs/en/setup)
- [Use Claude Code in VS Code (Claude Code docs)](https://code.claude.com/docs/en/vs-code)
- [Troubleshoot installation and login (Claude Code docs)](https://code.claude.com/docs/en/troubleshoot-install)
- [Connect Claude Code to tools via MCP (Claude Code docs)](https://code.claude.com/docs/en/mcp)
- [All settings (Claude Code docs)](https://code.claude.com/docs/en/settings-reference)
- [Add and manage MCP servers in VS Code (VS Code docs)](https://code.visualstudio.com/docs/copilot/chat/mcp-servers)
