# agentbox: a minimal default-deny sandbox for AI coding agents

This is one Bash script, `agentbox`. It needs no daemon, no VM, no container image, no Node and no Podman.

| Backend | Needs | Used when |
|---|---|---|
| **bwrap** (preferred) | `bubblewrap`, unprivileged user namespaces | available (default Ubuntu 22.04; 24.04 needs an AppArmor rule, see below) |
| **landlock** (fallback) | `python3`, kernel ≥ 5.13 | bwrap missing or user namespaces blocked. No root needed. |
| neither | n/a | the script **refuses to run**; it never falls back to running unsandboxed |

Inside the sandbox an agent sees:

- the project (read-write)
- the paths you grant explicitly (`--ro` / `--rw`)
- a **per-project agent home** (`~/.local/share/agentbox/<project>-<hash>/home`), mounted at `$HOME`
- read-only `/usr` and `/etc`

Your real `$HOME` (including `~/.ssh`, `~/.aws`, browser profiles and other repos) is either absent (bwrap) or unreadable (landlock). The environment is rebuilt from a whitelist, so `SSH_AUTH_SOCK`, `DBUS_SESSION_BUS_ADDRESS`, `AWS_*` and `*_TOKEN` variables are not inherited.

Grants are command-line arguments chosen by the human. The agent cannot edit them, because the launcher and its config live outside the sandbox.

## The four scenarios

| | **sudo** | **no sudo** |
|---|---|---|
| **Local laptop** | `sudo apt install bubblewrap` (+ 24.04 AppArmor rule). Optional: a VM for the highest assurance. | Use the bwrap that ships on Ubuntu Desktop. If user namespaces are blocked, only CLI agents work (landlock). |
| **Remote (Remote-SSH)** | Same as no-sudo, but create a **dedicated `agent` account** for the key (closes same-UID side channels). | Put `agentbox` in `~/.local/bin` and add one `authorized_keys` line with a forced command. Every session with that key is sandboxed. |

### Local: CLI agents (Claude Code, Codex)

```bash
cp agentbox ~/.local/bin/
agentbox check                         # must print PASS
agentbox ~/projects/app                # sandboxed shell; install agents *inside* it:
#   curl -fsSL https://claude.ai/install.sh | bash      (lands in the agent home)
agentbox ~/projects/app -- claude
agentbox --ro /data/reference ~/projects/app -- codex   # explicit extra grant
```

### Local: VS Code with Claude/Codex extensions

You install VS Code and its extensions once, and every sandbox shares them:

```bash
agentbox setup                                             # VS Code tarball -> ~/.local/share/agentbox/vscode (no sudo)
agentbox ext --install-extension anthropic.claude-code     # extension admin sandbox: sees no project
agentbox ext --install-extension openai.chatgpt
agentbox ext --update-extensions                           # update all extensions (you decide when)
agentbox code ~/projects/app                               # one sandboxed window per project
```

Both directories are mounted **read-only** at `/opt/vscode` and `/opt/vscode-extensions` in every project sandbox. An agent can't modify VS Code or an extension to attack your other projects, and in-sandbox auto-updates are switched off. Logins and settings stay per profile, as described under "Profiles" below. Your existing (snap) VS Code is untouched; keep it for work without AI.

This runs the whole VS Code process tree inside the sandbox: extension hosts, terminals, language servers and MCP servers. VS Code gets its own profile in the agent home, uses `--password-store=basic` (no access to the host keyring), and connects only to the Wayland socket. It never gets the D-Bus session bus.

Requirements:

- `agentbox setup` (or a system `.deb` VS Code). A snap cannot run under bwrap, because it needs to gain privileges and bwrap forbids that.
- **bwrap.** Landlock cannot hide the D-Bus or keyring sockets, so `code` mode refuses to use it.

Sign in to extensions inside the sandbox. See "Day to day" below for extensions, logins and git.

### Remote: VS Code Remote-SSH (works with or without sudo on the remote)

The remote host needs no container or second sshd. A dedicated SSH key is restricted by `authorized_keys`, so **everything** started with that key runs in `agentbox gate`: VS Code Server, remote extensions, terminals and agents.

On the **remote** host:

```bash
cp agentbox ~/.local/bin/ && ~/.local/bin/agentbox check     # must print PASS
# one line in ~/.ssh/authorized_keys per project (all on one line):
restrict,pty,port-forwarding,permitopen="localhost:*",permitopen="127.0.0.1:*",command="/home/me/.local/bin/agentbox gate /srv/work/app" ssh-ed25519 AAAA... agentbox-app
```

Grants go after the project path, for example `command=".../agentbox gate /srv/work/app --ro /data/ref"`. To revoke access, delete the line.

On the **laptop**, the VS Code that connects must itself be sandboxed, and only that sandbox may hold the key:

```bash
mkdir -p ~/agentbox-remote/app
agentbox ~/agentbox-remote/app -- ssh-keygen -t ed25519 -f ~/.ssh/id_agentbox   # key lives in agent home
agentbox ~/agentbox-remote/app -- cat ~/.ssh/id_agentbox.pub                   # paste into remote line above
agentbox ~/agentbox-remote/app -- sh -c 'cat >>~/.ssh/config' <<'EOF'
Host app-sandbox
  HostName remote.example.org
  User me
  IdentityFile ~/.ssh/id_agentbox
  IdentitiesOnly yes
  ForwardAgent no
EOF
agentbox code ~/agentbox-remote/app       # then: Remote-SSH -> app-sandbox
```

Even if the laptop-side agent steals `id_agentbox`, the key only opens the sandbox on the remote. Your real SSH key and `ssh-agent` never enter the sandbox. You don't need a tunnel helper.

### Profiles: which repos share a window

Each sandbox is one VS Code window. Repos in the same sandbox can see each other, and repos in different sandboxes cannot. Group repos by trust level:

```bash
agentbox code --rw ~/repos/mlengine --rw ~/repos/byprod ~/repos/main   # 3 repos, 1 window
AGENTBOX_PROFILE=work agentbox code ~/repos/byprod                       # share logins and settings with other "work" launches
```

Projects that share a profile also share a writable agent home. Something planted there in one project, such as a shell rc file, can affect the others, so only share profiles inside one trust group.

### Day to day: extensions, logins, git

**Rule of thumb: anything you log into inside a sandbox, the agent in that sandbox can use.** Logins are saved as files in the profile's agent home (`--password-store=basic`), and the agent can read them.

**Adding an extension while a window is open.** The Extensions view can browse but not install, because the folder is read-only. That's on purpose: otherwise an agent could install extensions too.

1. In the sandboxed window, right-click the extension and choose **Copy Extension ID**.
2. In a normal terminal *outside* the sandbox, run:

   ```bash
   agentbox ext --install-extension ms-python.python
   ```

3. In the window, run **Developer: Reload Window**. Every other sandbox picks the extension up on its next reload.

The terminal inside the sandbox can't run `agentbox ext`.

**Signing in (Claude, Codex, GitHub).** Use the extension's login or the Accounts icon in the bottom-left. The sandbox can't open your browser, so copy the login URL into your normal browser. The redirect returns to a localhost server that VS Code starts, and the sandbox shares the network, so this works. Each profile signs in once. (GitHub sign-in is untested.)

**GitHub and git push.** A normal VS Code GitHub login gives access to *all* your repos, which undoes the separation between projects. Prefer, in this order:

1. **Push from outside the sandbox.** The agent commits, and you run `git push` in a normal terminal. The agent never holds a credential.
2. **A fine-grained GitHub token for this one repo** (GitHub → Settings → Developer settings → Fine-grained tokens):

   ```bash
   agentbox ~/repos/app -- sh -c 'git config --global credential.helper store
     git config --global user.name "Your Name"; git config --global user.email you@company.com'
   ```

   The first `git push` inside the sandbox asks for the token once. If it leaks, only that repo is exposed.
3. **A full GitHub login.** Only for profiles whose repos all belong to the same trust group.

Your normal `~/.gitconfig`, `~/.ssh` and `gh` login are invisible in the sandbox, so set your git name and email per profile as above.

**Avoid Settings Sync** in sandboxes. It would pull your normal profile, including its extension list and settings, into the sandbox.

**Data outside the project.** Grant it explicitly and read-only when possible:

```bash
agentbox code --ro /data/reference ~/repos/app
```

To revoke a grant, close the window and relaunch without it.

### What sudo adds

- **Ubuntu 24.04 user-namespace restriction** (`kernel.apparmor_restrict_unprivileged_userns=1`). If `agentbox check` falls back to landlock, allow bwrap once:

  ```bash
  sudo tee /etc/apparmor.d/bwrap <<'EOF'
  abi <abi/4.0>,
  include <tunables/global>
  profile bwrap /usr/bin/bwrap flags=(unconfined) { userns, }
  EOF
  sudo apparmor_parser -r /etc/apparmor.d/bwrap
  ```

- **No bwrap and no sudo:** `apt-get download bubblewrap && dpkg-deb -x bubblewrap_*.deb ~/.local/bwrap`, then add `~/.local/bwrap/usr/bin` to `PATH`. This works only where user namespaces are allowed; otherwise landlock is used.
- **Dedicated remote account** (recommended on shared servers):

  ```bash
  sudo useradd -m agent
  sudo setfacl -R -m u:agent:rwX /srv/work/app
  ```

  Put the `authorized_keys` line in `~agent/.ssh/`. A separate UID also blocks what namespaces can't: signals and abstract sockets aimed at your own processes, `/tmp` files of your UID, and the kernel keyring.
- **Highest assurance on the laptop:** run the same `agentbox` inside a VM. This adds protection against kernel exploits. It isn't required for the filesystem boundary.

## Verification

`agentbox check [DIR]` places a canary file in your real home and sets a canary environment variable. It then asserts, from inside the sandbox, that:

- the canary file, `~/.ssh` and `/root` are unreadable
- the environment and `SSH_AUTH_SOCK` are not inherited
- the project is writable

If any assertion fails, it exits non-zero.

Tested on Ubuntu 22.04, kernel 6.8, bwrap 0.6.1:

- `check` passes with **both** backends. The same probes fail when run unsandboxed.
- Gate end to end through an unprivileged sshd: commands, stdin scripts (the Remote-SSH style), sandboxed `$HOME`, and a forward to `127.0.0.1` all work. Forwarding to `1.1.1.1` is denied by `permitopen`.
- `setup` installed VS Code 1.141.2, and `ext` installed Claude Code into the shared folder.
- `agentbox code` started the GUI under Wayland inside the sandbox. VS Code's attempt to auto-update an extension failed against the read-only mount, as intended.
- **Not tested yet:** using the Claude/Codex extensions interactively, a full VS Code Remote-SSH session through the gate, and Ubuntu 24.04.

## Known limits (be honest about these)

- **Network is open.** Agents need their API. A sandboxed process can reach anything the host can reach, including localhost services. Egress filtering needs a proxy or firewall, which means root.
- **Same UID (no-sudo mode).**
  - The bwrap backend hides other processes (PID namespace), but abstract Unix sockets are shared through the network namespace. This includes X11's `@/tmp/.X11-unix`, so prefer Wayland.
  - The landlock backend additionally leaves path-based Unix sockets (`/run/user/$UID/bus` and keyring) connectable, and signals are unscoped below kernel 6.12.
  - Use a dedicated account where you can.
- **Remote-SSH server lifetime.** With `--unshare-pid`, the VS Code Server lives as long as the SSH session that started it, so a reconnect restarts it.
- **Anything you grant is fully exposed.** That includes secrets inside the project (`.env`) and anything the agent logs into.
- **Container- and kernel-escape bugs** are out of scope without a VM.
