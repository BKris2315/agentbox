# agentbox: a minimal default-deny sandbox for AI coding agents

This is one Bash script, `agentbox`. It needs no daemon, no VM, no container image, no Node and no Podman, only `/usr/bin/python3` (preinstalled on Ubuntu).

| Backend | Needs | Used when |
|---|---|---|
| **bwrap** (preferred) | `bubblewrap`, unprivileged user namespaces | available (default Ubuntu 22.04; 24.04 needs an AppArmor rule, see below) |
| **landlock** (fallback) | kernel ≥ 5.13 | bwrap missing or user namespaces blocked. No root needed. |
| neither | n/a | the script **refuses to run**; it never falls back to running unsandboxed |

Inside the sandbox an agent sees:

- the project (read-write, except `.git/config` and `.git/hooks`, see below)
- the paths you grant explicitly (`--ro` / `--rw`), minus anything you `--mask`
- a **per-project agent home** (`~/.local/share/agentbox/<project>-<hash>/home`), mounted at `$HOME`
- read-only `/usr` and `/etc`

Your real `$HOME` (including `~/.ssh`, `~/.aws`, browser profiles, other repos and your personal folders) is either absent (bwrap) or unreadable (landlock). The environment is rebuilt from a whitelist, so `SSH_AUTH_SOCK`, `DBUS_SESSION_BUS_ADDRESS`, `AWS_*` and `*_TOKEN` variables are not inherited.

Grants are command-line arguments chosen by the human. The agent cannot edit them, because the launcher and its config live outside the sandbox.

## How it is enforced

The sandbox is enforced by the kernel, not by watching commands. A Bash script, a Python script or a compiled binary the agent writes is confined exactly like the agent itself.

### Layers

| Layer | What it stops | Backend |
|---|---|---|
| Mount namespace / Landlock filesystem rules | reading or writing anything outside the grants | both |
| Protected paths | granting home, secrets or personal folders by mistake | both |
| Read-only `.git/config` and `.git/hooks`, pinned `.git` | planting code that git later runs **on the host** | bwrap |
| `--mask` | secrets inside a granted tree (`.env`) | bwrap |
| seccomp filter | kernel keyring, ptrace, new namespaces, io_uring, bpf, mount, TIOCSTI keystroke injection | both |
| Landlock TCP port rules (`--net web`) | local databases, Docker API, dev servers on non-ephemeral ports | both, kernel ≥ 6.7 |
| No network + allowlisting proxy (`--net proxy`) | exfiltration to arbitrary hosts, host abstract Unix sockets (X11) | bwrap |
| Dedicated account (`agentbox adduser`) | same-UID side channels | both, needs sudo |
| VM | kernel exploits | any |

### Protected paths

Some paths can never be granted: your home itself (or `/`), agentbox's own state, credential stores (`~/.ssh`, `~/.gnupg`, `~/.aws`, `~/.kube`, `~/.docker`, `~/.config`, keyrings, `.netrc`, `.git-credentials`, ...) and files your host executes (`~/.bashrc`, `~/.profile`, `~/.local/bin`, `~/bin`). This applies to the path itself, anything inside it, and anything that contains it, so `--rw ~/.local` is refused too.

**Personal folders** (Desktop, Documents, Downloads, Music, Pictures, Videos, Public, Templates, plus any locations set in `~/.config/user-dirs.dirs`) are protected by default. A whole personal folder is never granted. To grant a subfolder, pass `--allow-personal`:

```bash
agentbox --allow-personal --ro ~/Documents/specs ~/repos/app
```

### Git: code that runs outside the sandbox

Git runs commands named in `.git/config` (`core.fsmonitor`, `core.sshCommand`, filter drivers, ...) and scripts in `.git/hooks` with **your full rights** whenever you use the repo outside the sandbox. A plain `git status` is enough to trigger some of them. Under bwrap, agentbox therefore:

- mounts `.git/config`, `.git/hooks` and `.git/info` read-only, and does the same for submodules (`.git/modules/*`), for every repo up to 3 levels deep in the project and `--rw` grants
- bind-mounts `.git` onto itself, so the agent cannot rename it away and plant a fake one

The agent can still commit, branch and rebase. To change the repo config, do it outside the sandbox.

The landlock backend can't make part of a granted tree read-only, so it prints a warning. Either way, use `agentbox git` for git commands outside the sandbox on agent-touched repos:

```bash
agentbox git -C ~/repos/app push
```

It refuses to run if the repo config contains anything that runs commands or redirects git (`core.fsmonitor`, `core.sshCommand`, `alias.*`, `url.*`, `include*`, filter or diff drivers, ...), and it never runs hooks.

Other files a host tool may execute: `Makefile`, `package.json` scripts, `.envrc` (direnv) and `.vscode/tasks.json`. Review the diff before running them outside the sandbox, and keep VS Code's Workspace Trust enabled for agent-touched repos.

### Network modes (`--net`)

| Mode | Behaviour | Use for |
|---|---|---|
| `auto` (default) | `web` on kernel ≥ 6.7, otherwise `open` with a warning | |
| `web` | TCP connect only to ports 443, 80, 53, 22, the ephemeral range (`ip_local_port_range`, used by IDE, debugger and forwarded ports) and `--port N[-M]` grants. Fails closed on older kernels. | everyday use |
| `proxy` | No network except an HTTPS proxy (bwrap only). The proxy runs outside the sandbox and allows only the hosts in `~/.local/share/agentbox/allowlist` plus `--allow-host`. `HTTPS_PROXY` is set inside the sandbox. | untrusted code, sensitive repos |
| `open` | Same network as the host | Remote-SSH from a sandboxed VS Code, anything `web` breaks |

`web` blocks connections to services on ports such as 5432, 6379, 2375, 3000 and 8080, on any host. If the agent needs to reach its own dev server, add `--port 3000`.

`proxy` notes:

- It writes a default allowlist on first use (Anthropic, OpenAI, GitHub, npm, PyPI, the VS Code marketplace). Edit the file from outside the sandbox; changes apply to new connections immediately.
- Decisions are logged to `~/.local/share/agentbox/<profile>/proxy.log`. Check it when something doesn't connect.
- Only HTTPS `CONNECT` is proxied. Plain HTTP, SSH and raw TCP don't work.
- The sandbox has its own network namespace, so the localhost callback used by some logins can't reach it. Log in once with `--net web` using the same profile.

```bash
agentbox --net proxy --allow-host gitlab.example.com ~/repos/app -- claude
```

### Masking secrets in a grant

```bash
agentbox --mask ~/repos/app/.env --mask ~/repos/app/secrets ~/repos/app -- claude
```

Inside the sandbox, a masked file appears empty and a masked directory appears empty and read-only (bwrap only).

### seccomp filter

Every sandbox runs under a seccomp filter. It returns `EPERM` for:

- `keyctl`, `add_key`, `request_key` (your kernel keyring)
- `ptrace` and `process_vm_*`, so `gdb` and `strace` don't work inside
- creating namespaces (`unshare`, `clone` with namespace flags, `setns`), so tools that sandbox themselves with bwrap don't work inside
- `io_uring`, `bpf`, `userfaultfd`, `perf_event_open`, mount-related calls, kernel modules, `kexec`
- the `TIOCSTI` and `TIOCLINUX` ioctls

`agentbox code` uses a variant that still allows namespaces, because VS Code's Chromium sandbox needs them.

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
agentbox code ~/agentbox-remote/app       # then: Remote-SSH -> app-sandbox (if it fails to connect: add --net open)
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

**Signing in (Claude, Codex, GitHub).** Use the extension's login or the Accounts icon in the bottom-left. The sandbox can't open your browser, so copy the login URL into your normal browser. The redirect returns to a localhost server that VS Code starts. That works with `--net web` and `open`, because the sandbox shares the host network, but not with `--net proxy`. Each profile signs in once, so log in with the default mode first. (GitHub sign-in is untested.)

**GitHub and git push.** A normal VS Code GitHub login gives access to *all* your repos, which undoes the separation between projects. Prefer, in this order:

1. **Push from outside the sandbox.** The agent commits, and you run `agentbox git -C ~/repos/app push` in a normal terminal. The agent never holds a credential, and repo config or hooks the agent may have planted never run with your rights (see "Git" above).
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
- **Dedicated account** (recommended on shared servers, useful on laptops too):

  ```bash
  agentbox adduser agent ~/repos/app /srv/work/other   # useradd + ACLs + agentbox check, via sudo
  sudo -u agent -i agentbox ~/repos/app -- claude       # local use
  ```

  It creates the account and installs agentbox in its `~/.local/bin`. It then grants the account read-write ACLs on each project, adds default ACLs so you and the agent can both use each other's new files, and gives traverse-only (`--x`) access on the parent directories. Finally it runs `agentbox check` as the new user. For remote use, put the `authorized_keys` line in `~agent/.ssh/` with the gate path `~agent/.local/bin/agentbox`.

  A separate UID also blocks what namespaces and seccomp can't: signals and abstract sockets aimed at your own processes, `/tmp` files of your UID, and everything readable only by you.

### Highest assurance: a VM

Every layer above shares the host kernel, so a kernel exploit defeats them all. If that matters, run the same `agentbox` inside a lightweight VM, for example [Multipass](https://multipass.run/) or [Lima](https://lima-vm.io/):

```bash
multipass launch 24.04 --name agents --cpus 4 --memory 8G --disk 40G
multipass mount ~/repos/app agents:/home/ubuntu/app      # share only the project
multipass shell agents                                    # then inside the VM:
#   install agentbox, run agentbox check, then agentbox ~/app -- claude
```

Share only project directories into the VM, never your home. Running agentbox inside the VM still matters: it keeps the agent away from the VM's own credentials and from other projects shared into it.

## Verification

`agentbox check [DIR]` tries to break out the way an agent's script would. It runs as a Python program inside the sandbox, once per network mode (`web`, and `proxy` under bwrap):

- **Grants (host side):** granting your home, `~/.ssh`, `~/.config`, `~/.local/bin`, `~/.bashrc` or any personal folder is refused.
- **Files:** a canary in your real home, `~/.ssh`, every personal folder and `/root` are unreadable.
- **Environment:** a canary variable and `SSH_AUTH_SOCK` are not inherited.
- **Processes:** the host's `/proc/<pid>/environ` is unreadable.
- **seccomp:** `keyctl`, `ptrace`, `io_uring`, creating a user namespace and `TIOCSTI` all fail with `EPERM`. The probes are built so the kernel's own errors can't be mistaken for a pass.
- **Git:** on a fixture repo, `.git/config` and `.git/hooks` can't be written and `.git` can't be renamed, but committing still works.
- **Mask:** a masked `.env` reads empty.
- **Network:**
  - `web`: a listener on a non-allowed local port can't be reached.
  - `proxy`: direct internet is unreachable, the proxy refuses a host that isn't on the allowlist, and a host abstract Unix socket can't be reached.
- **Project:** the project is writable.

Lines marked `limit` are documented gaps of the current backend or mode (see "Known limits"). They don't count as failures. Any `FAIL` exits non-zero.

Tested on Ubuntu 22.04, kernel 6.8 (Landlock ABI 4), bwrap 0.6.1:

- `check` passes with **both** backends. Run unsandboxed, the same probes report `FAIL`.
- With `--net proxy`, `curl https://api.github.com` succeeds through the proxy and `https://example.com` is refused. Both are logged.
- A `python3` planted in the agent home's `~/.local/bin` is not picked up by the next launch. Earlier versions ran it **outside** the sandbox on the landlock backend.
- `agentbox git` refuses a repo with `core.fsmonitor` or a filter driver in its config.
- `ext --list-extensions` works under the seccomp filter, and the `gui` filter variant allows namespaces while still blocking the keyring.
- Gate end to end through an unprivileged sshd: commands, stdin scripts (the Remote-SSH style), sandboxed `$HOME`, and a forward to `127.0.0.1` all work. Forwarding to `1.1.1.1` is denied by `permitopen`.
- `setup` installed VS Code 1.141.2, and `ext` installed Claude Code into the shared folder.
- `agentbox code` started the GUI under Wayland inside the sandbox. VS Code's attempt to auto-update an extension failed against the read-only mount, as intended.
- **Not tested yet:**
  - launching the VS Code GUI under the seccomp filter and `--net web`
  - using the Claude/Codex extensions interactively
  - a full VS Code Remote-SSH session (through the gate, or from a sandboxed laptop VS Code under `--net web`; fall back to `--net open`)
  - `agentbox adduser`
  - aarch64
  - Ubuntu 24.04

## Known limits

- **Network, except with `--net proxy`.**
  - `web` restricts ports, not hosts. The agent can send anything it can read to any server on port 443.
  - `web` also allows the whole ephemeral port range and port 22 on every host, localhost included.
  - Kernels older than 6.7 get `open`.
  - Even `proxy` lets the agent send data to the hosts you allowlist, for example a GitHub gist.
- **Same UID (no-sudo mode).**
  - The bwrap backend hides other processes (PID namespace), but abstract Unix sockets are shared through the network namespace, except with `--net proxy`. This includes X11's `@/tmp/.X11-unix`, so prefer Wayland.
  - The landlock backend additionally leaves path-based Unix sockets (`/run/user/$UID/bus` and keyring) connectable, and signals are unscoped below kernel 6.12 (Landlock ABI 6 scopes both automatically).
  - Use a dedicated account (`agentbox adduser`) where you can.
- **Git and other host-executed files on the landlock backend.** `.git/config` and `.git/hooks` stay writable. Use `agentbox git`. On both backends, review `Makefile`, `package.json` scripts, `.envrc` and `.vscode/tasks.json` before running them outside the sandbox.
- **seccomp trade-offs.** `gdb`, `strace` and tools that build their own namespace sandbox (including nested bwrap) don't work inside. The `gui` variant used by `agentbox code` allows namespaces, which exposes more of the kernel.
- **Remote-SSH server lifetime.** With `--unshare-pid`, the VS Code Server lives as long as the SSH session that started it, so a reconnect restarts it.
- **Anything you grant is fully exposed.** That includes secrets inside the project that you didn't `--mask`, and anything the agent logs into. `--mask` needs bwrap.
- **Container- and kernel-escape bugs** are out of scope without a VM.
