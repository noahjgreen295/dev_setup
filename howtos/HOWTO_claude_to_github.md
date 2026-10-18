# HOWTO: Integrate Claude Code with GitHub

How to give Claude GitHub access from three places: Claude chat, Claude Code in VS Code on WSL, and Claude Code on the WSL command line. Full access here means the permissions set on the token in Preliminaries. Do Preliminaries first, then the section for each place you use.

## Preliminaries

Each place gets GitHub access its own way, and none of them shares it with the others:

- **Chat** uses the Claude GitHub App, installed on your GitHub account. It doesn't use the token.
- **Claude Code in VS Code and on the WSL command line** use Anthropic's GitHub plugin, which sends your personal access token to GitHub. They can share the plugin, but each needs the token set in its own place.

Connecting GitHub in chat does not give Claude Code GitHub access. Chat connectors that are MCP servers, such as Google Calendar, carry over to Claude Code automatically, but GitHub in chat is an app, not an MCP server.

### Create the personal access token

1. Go to github.com/settings/personal-access-tokens and create a fine-grained token.
2. Under repository access, choose All repositories, or pick the repositories Claude should reach. If you pick specific ones, add each new repository to the token as you create it. The token value doesn't change when you do.
3. Set each permission listed below to Read and write.
4. Generate the token and copy it right away. GitHub shows it only once.

Permissions:

- **Actions:** lets Claude see workflow runs and why CI failed, trigger runs manually, and turn workflows on or off.
- **Contents:** lets Claude read and push code.
- **Issues:** lets Claude read, create and update issues.
- **Pull requests:** lets Claude read, open and update pull requests.
- **Secrets (optional):** lets Claude set repository secrets that workflows use. Leave it off if you would rather add secrets yourself in the repository settings.
- **Variables (optional):** lets Claude set non-secret workflow settings.
- **Workflows:** lets Claude create or change files in `.github/workflows`. GitHub requires this in addition to Contents, so CI changes fail without it.

A token with no permissions selected can only read public repositories. To change permissions later, edit the token on the same GitHub page; the token value stays the same.

## Chat

Chat needs two things: your GitHub account linked to Claude, and the Claude GitHub App installed on your GitHub account. The link alone lets Claude see your repositories but not change them.

1. **Link your account.** In claude.ai, open the connectors settings at claude.ai/customize/connectors and connect GitHub.
2. **Install the app.** Go to github.com/apps/claude/installations/select_target and choose your personal account. The page is titled Install & Authorize Claude.
3. **Choose repositories.** Pick All repositories, or Only select repositories and choose the ones Claude should work in. Then click Install & Authorize.
4. **Confirm.** Start a new chat and ask Claude to list your repositories and check that it can push to one.

The app's permissions come from GitHub's install screen, not from your token: read access to administration, commit statuses, merge queues and metadata, and read and write access to actions, checks, code, discussions, issues, pull requests, repository hooks and workflows. That covers everything in the token except secrets and variables.

### Troubleshooting

- **Claude can see a repository but its pushes are refused.** The app isn't installed, or it doesn't cover that repository. If you chose Only select repositories, add the repository on GitHub under Settings, Applications, Installed GitHub Apps, then Configure next to Claude. New repositories aren't added automatically.

## WSL VS Code

1. **Install the plugin.** In the Claude Code panel, type `/plugins`, find **github** in the official marketplace (`claude-plugins-official`), and install it. Don't click the plugin's Authenticate button: that browser sign-in doesn't work with Claude Code, and the token replaces it.
2. **Give it the token.** Press Ctrl+Shift+P, run Preferences: Open User Settings (JSON), and add this setting. Replace `github_pat_...` with your full token, keeping the quotes, then save the file.

    ```
    "claudeCode.environmentVariables": [{ "name": "GITHUB_PERSONAL_ACCESS_TOKEN", "value": "github_pat_..." }]
    ```

3. **Reload.** Run Developer: Reload Window from the Command Palette, then start a new Claude session.
4. **Confirm.** Type `/mcp` in the Claude panel. GitHub should show as connected.

Set the token in this VS Code setting, not in `~/.zshrc`: VS Code doesn't always pass your shell's environment to Claude Code. The variable name must be exactly `GITHUB_PERSONAL_ACCESS_TOKEN`, which is what the plugin's config file sends to GitHub as a bearer token.

### Troubleshooting

- **“Authorization header is badly formatted.”** Claude Code didn't receive the token. Check the setting's name is exactly `GITHUB_PERSONAL_ACCESS_TOKEN`, then reload the window and start a new session.
- **401 from GitHub.** The token value is wrong or expired. Check that the setting holds your full token, not the `github_pat_...` placeholder. If the token has expired, regenerate it on GitHub and paste the new value.

## WSL command line

Claude Code on the command line reads the token from your shell, not from the VS Code setting, so it needs its own copy.

1. **Check the plugin.** In a WSL terminal, run `claude` and type `/plugin`. If github isn't listed as installed, run `/plugin install github@claude-plugins-official`.
2. **Add the token.** Open `~/.zshrc` in WSL and add this line at the bottom, with your full token in place of the placeholder:

    ```
    export GITHUB_PERSONAL_ACCESS_TOKEN="github_pat_..."
    ```

3. **Reload the shell.** Open a new terminal window, then run `claude`.
4. **Confirm.** Type `/mcp`. GitHub should show as connected.

### Troubleshooting

- **“Authorization header is badly formatted.”** The shell didn't have the token when you started claude. Run `echo ${GITHUB_PERSONAL_ACCESS_TOKEN:0:11}`; it should print `github_pat_`. If it prints nothing, check the line in `~/.zshrc` and open a new terminal.
- **401 from GitHub.** Same as in VS Code: the value in `~/.zshrc` is wrong, still the placeholder, or expired.

## Sources

- [Connect Claude Code to tools via MCP (Claude Code docs)](https://code.claude.com/docs/en/mcp)
- [Use Claude Code in VS Code (Claude Code docs)](https://code.claude.com/docs/en/vs-code)
- [Use Claude Code in the cloud (Claude Code docs)](https://code.claude.com/docs/en/claude-code-on-the-web)
- [GitHub plugin config file (.mcp.json)](https://raw.githubusercontent.com/anthropics/claude-plugins-official/main/external_plugins/github/.mcp.json)
- [GitHub plugin authentication issue, claude-plugins-official #285](https://github.com/anthropics/claude-plugins-official/issues/285)
- [Managing your personal access tokens (GitHub Docs)](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)
- [REST API endpoints for repository contents (GitHub Docs)](https://docs.github.com/en/rest/repos/contents)
- [REST API endpoints for workflows (GitHub Docs)](https://docs.github.com/en/rest/actions/workflows)
