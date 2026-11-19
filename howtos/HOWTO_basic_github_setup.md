# HOWTO: Basic GitHub setup

How to set up a computer so `git` can push to and pull from GitHub on the command line without asking you to log in each time. Written for WSL (Ubuntu); the same steps work on Linux, and on a Mac with the changes below.

## On a Mac

Everything is the same except steps 3 and 4:

- **Step 3: no `keychain`.** macOS runs an SSH agent for you and can keep the passphrase in the Mac's Keychain, so skip installing `keychain` and don't add its line to `~/.zshrc`. Instead, add `UseKeychain yes` to the `~/.ssh/config` block (leave it out if the key has no passphrase):

    ```
    Host github.com
      HostName github.com
      User git
      IdentityFile ~/.ssh/id_ed25519
      AddKeysToAgent yes
      UseKeychain yes
    ```

    Then store the passphrase once:

    ```
    ssh-add --apple-use-keychain ~/.ssh/id_ed25519
    ```

    Use the `ssh-add` that comes with macOS, not one from Homebrew; only Apple's has `--apple-use-keychain`. After this, no passphrase prompts, even after a restart.

- **Step 4: copying the key.** `pbcopy < ~/.ssh/id_ed25519.pub` puts the public key on the clipboard.

A Mac has only one `~/.ssh`, so the note below about WSL and Windows keeping separate keys doesn't apply.

## Tell git who you are

Every commit records a name and email. Until you set them, git refuses to commit and asks you to. Set them once per environment (WSL and Windows each keep their own):

```
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
```

- **Email.** Use an email address on your GitHub account, so GitHub links the commits to you. To keep your real address out of public commit history, go to GitHub **Settings** > **Emails**, tick **Keep my email addresses private**, and use the `...@users.noreply.github.com` address it shows.
- **Default branch.** New repos made with `git init` start on `main`, matching what GitHub uses, instead of `master`.

Check them with `git config --global --list`. They're stored in `~/.gitconfig`.

## SSH keys

You make a key pair on your computer. The private key stays on the computer; the public key goes on GitHub. When git talks to GitHub over SSH, the two are matched and no password is needed.

Keys belong to one environment. WSL and Windows each have their own `~/.ssh`, so a key made in WSL doesn't help Git for Windows, and the reverse. Do these steps in each place you use git.

### 1. Check for an existing key

```
ls ~/.ssh
```

If you see `id_ed25519` and `id_ed25519.pub`, you already have a key. Skip to step 3, or make a new one if you don't remember its passphrase.

### 2. Make a key

```
ssh-keygen -t ed25519 -C "you@example.com"
```

- Use your GitHub email address. It's only a label.
- Press Enter to accept the default file location.
- Enter a passphrase. It protects the key if the computer is stolen; step 3 means you type it once per session rather than on every push. An empty passphrase works too, with no prompts at all, but anyone with the file can use it.

### 3. Have the agent remember the passphrase

An agent holds the unlocked key in memory. WSL doesn't start one on its own, and a plain `ssh-agent` would only serve one terminal. `keychain` starts one agent and shares it across all terminals. Install it:

```
sudo apt install keychain
```

Add this line to the end of `~/.zshrc`:

```
eval "$(keychain --eval --quiet id_ed25519)"
```

Then create or edit `~/.ssh/config` so SSH uses this key for GitHub:

```
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519
  AddKeysToAgent yes
```

Open a new terminal. It asks for the passphrase once; after that, no prompts until you restart WSL or the computer.

### 4. Add the public key to GitHub

Print the public key and copy all of it:

```
cat ~/.ssh/id_ed25519.pub
```

On GitHub, go to your profile picture > **Settings** > **SSH and GPG keys** > **New SSH key**. Give it a title naming the computer (e.g. "Laptop WSL"), leave Key type as **Authentication Key**, paste the key, and click **Add SSH key**.

Never paste or share the file without `.pub`. That's the private key.

### 5. Test the connection

```
ssh -T git@github.com
```

The first time, it asks whether to trust GitHub's host key. Type `yes`. You should see:

```
Hi <username>! You've successfully authenticated, but GitHub does not provide shell access.
```

### 6. Use SSH URLs for repos

Clone with the SSH URL (the **SSH** tab under the green **Code** button):

```
git clone git@github.com:<owner>/<repo>.git
```

A repo you already cloned with an `https://` URL keeps asking for credentials. Switch it to SSH from inside the repo:

```
git remote -v
git remote set-url origin git@github.com:<owner>/<repo>.git
```

## Troubleshooting

- **`Permission denied (publickey)`.** The key isn't on GitHub, or SSH isn't offering it. Check `ssh-add -l` lists the key; if not, run `ssh-add ~/.ssh/id_ed25519`. Make sure the key on GitHub matches `~/.ssh/id_ed25519.pub`.
- **Still asked for a username and password.** The repo's remote is HTTPS. See step 6.
- **Asked for the passphrase in every terminal.** Check the `keychain` line is in `~/.zshrc` and that `keychain` is installed (`which keychain`).
- **`bad permissions` / `UNPROTECTED PRIVATE KEY FILE`.** Run `chmod 700 ~/.ssh && chmod 600 ~/.ssh/id_ed25519`.

## Sources

- [First-time Git setup (Pro Git book)](https://git-scm.com/book/en/v2/Getting-Started-First-Time-Git-Setup)
- [Setting your commit email address (GitHub docs)](https://docs.github.com/en/account-and-profile/setting-up-and-managing-your-personal-account-on-github/managing-email-preferences/setting-your-commit-email-address)
- [Generating a new SSH key and adding it to the ssh-agent (GitHub docs)](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)
- [Adding a new SSH key to your GitHub account (GitHub docs)](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)
- [Testing your SSH connection (GitHub docs)](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/testing-your-ssh-connection)
- [Managing remote repositories (GitHub docs)](https://docs.github.com/en/get-started/getting-started-with-git/managing-remote-repositories)
