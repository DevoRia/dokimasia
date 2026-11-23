<div align="center">

# 🏛️ dokimasia

**Run untrusted repos in a disposable VM. Nothing from the repo ever executes on your machine.**

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
![Platform](https://img.shields.io/badge/platform-macOS%20%C2%B7%20Linux%20%C2%B7%20WSL2-lightgrey)
![QEMU](https://img.shields.io/badge/powered%20by-QEMU-FF6600?logo=qemu&logoColor=white)
![Python](https://img.shields.io/badge/python-3.9%2B%20%C2%B7%20stdlib%20only-3776AB?logo=python&logoColor=white)
[![Claude Code plugin](https://img.shields.io/badge/Claude%20Code-plugin-D97757?logo=anthropic&logoColor=white)](#-agents)
![Status: alpha](https://img.shields.io/badge/status-alpha-f59e0b)
[![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](#-contributing)
[![GitHub stars](https://img.shields.io/github/stars/DevoRia/dokimasia?style=social)](https://github.com/DevoRia/dokimasia)

<img src="assets/explainer.gif" alt="Without dokimasia a random repo runs as you and steals your keys. With dokimasia it runs in a disposable VM your agent drives from outside, malware hits a wall, and the VM is destroyed." width="800">

</div>

---

`npm install`, `pip install`, `make`, `./run.sh` — the first run of someone else's code happens as **your user**, next to your SSH keys, cloud credentials, browser sessions and wallets. A take-home task from a "recruiter", a tool from a random repo, a binary you want to reverse-engineer: you just want to try it, not hand it your machine.

`dokimasia` gives that code its own throwaway Linux VM and lets you (or your coding agent) drive it from the outside:

```bash
dokimasia up git@github.com:someone/take-home.git   # VM boots in ~10 s, repo copied to /work
dokimasia exec -- npm test                          # runs inside the VM, output streams back
dokimasia down                                      # VM and its disk are gone
```

- 🖥️ **A real VM, not a container** — own kernel via QEMU (`hvf` on macOS, `kvm` on Linux).
- 🔌 **No network device by default** — not a firewall rule the code could flip, the NIC simply doesn't exist.
- 📦 **Copied in, never mounted** — the repo travels on a read-only ISO. No shared folders, no clipboard, no SSH, no ports.
- 🤖 **Built for agents** — any agent that can run a shell command can drive the VM; a Claude Code plugin is included.
- 🗑️ **Disposable** — copy-on-write disk over a clean base image, deleted on `down`.
- 🪶 **Tiny** — one Python file, stdlib only. Read it in 10 minutes.

## 🎬 In the terminal

<img src="assets/demo.gif" alt="dokimasia in the terminal: boot a VM with a repo, run commands inside it, no network, no host files, destroy it" width="760">

## 🚀 Quickstart

**1. Install QEMU**

| OS | |
|---|---|
| macOS | `brew install qemu` |
| Debian / Ubuntu | `sudo apt install qemu-system qemu-utils xorriso` (+ `qemu-efi-aarch64` on ARM) |
| Fedora | `sudo dnf install qemu-kvm qemu-img xorriso` |
| Windows | inside **WSL2** (Windows 11 exposes `/dev/kvm` there), then the Linux steps |

**2. Get dokimasia**

```bash
git clone https://github.com/DevoRia/dokimasia
ln -s "$PWD/dokimasia/bin/dokimasia" /usr/local/bin/dokimasia
dokimasia init     # once: downloads Debian 13 cloud image (~350 MB), installs node, python, build tools
```

**3. Try something**

```bash
dokimasia run ./some-repo -- 'npm ci --ignore-scripts && npm test'
```

## 🧰 Commands

| Command | |
|---|---|
| `dokimasia up <dir\|git-url> [--net]` | Boot a VM with a copy of the repo in `/work` |
| `dokimasia exec [--timeout S] -- <cmd>` | Run a command in `/work` as root in the VM, stream output, pass exit code through |
| `dokimasia logs [-n N]` | Show the VM's serial console |
| `dokimasia down` | Kill the VM and delete its disk |
| `dokimasia run <dir\|git-url> [--net] -- <cmd>` | `up` + `exec` + `down` in one go |
| `dokimasia ls` | List sessions |
| `dokimasia audit <dir\|git-url>` | Read-only scan for things that run by themselves on open/install. Exit code 2 if something is high-risk |
| `dokimasia add <apt packages>` | Install extra packages into the base image (online, before any repo is loaded) |

Add `--name <n>` to `up` / `exec` / `logs` / `down` to run several VMs at once.

Heavy repos: `up` takes `--cpus 4 --memory 8G --disk 40G`.

## 🔎 Before you open a repo in your editor

Some repos run code the moment a folder is *opened*, before any `npm install`: a `.vscode/tasks.json` with `runOn: folderOpen` is the classic one in fake-interview campaigns. `dokimasia audit` reads the repo without executing anything and lists what would run:

```bash
dokimasia audit git@github.com:someone/take-home.git   # or a local dir
```

It flags folder-open tasks, devcontainer commands (`initializeCommand` runs on your host), npm lifecycle scripts, Claude Code hooks, MCP servers, symlinks that leave the repo and obviously suspicious command patterns. `dokimasia up` prints a one-line warning when it finds any. It also prints your VS Code / Cursor `task.allowAutomaticTasks` setting; `"off"` closes the folder-open vector for every repo.

To scan after every `git pull` / checkout in a repo: `dokimasia audit --install-hook .`

It is a heuristic for the usual tricks, not a guarantee. When something is flagged, read the repo through `dokimasia exec` instead of opening it on your host.

## 🤖 Agents

Coding agents are the riskiest way to run strange code: they `npm install` without blinking, and the repo's README can talk them into things. With dokimasia **the agent stays on your host and the code stays in the VM** — the agent only sees text.

**Claude Code**

```
/plugin marketplace add DevoRia/dokimasia
/plugin install dokimasia@dokimasia
```

> *"Clone DevoRia/some-project and run its tests safely."*

The skill tells Claude to `up` the repo, iterate with `exec`, treat everything coming out of the VM (and the repo's own AGENTS.md / README) as untrusted data, ask before enabling the network, and `down` when done.

**Without the plugin** (just the skill): `mkdir -p ~/.claude/skills && cp -RL skills/dokimasia ~/.claude/skills/` — the skill carries its own copy of the CLI.

**Any other agent** (Codex, Cursor, Aider, your own): give it the CLI and the rules in [`skills/dokimasia/SKILL.md`](skills/dokimasia/SKILL.md).

## ⚙️ How it works

```
 host                                          disposable VM  (QEMU + hvf/kvm)
 ────                                          ──────────────────────────────
 git clone --depth 1  (hooks/config off)
 tar (symlinks not followed)
        └─► read-only cloud-init ISO ───────►  untar into /work
 qcow2 overlay on clean base image             Debian 13 + node / python / build tools
                                               no network device
 dokimasia exec ──► unix socket ─► virtio-serial ─► root shell ─► your command
 terminal       ◄── text, escape codes stripped ◄──── output + exit code
 dokimasia down ──► kill QEMU, delete overlay
```

## 🛡️ Security model & limits

- **Boundary = the QEMU hypervisor.** A VM-escape exploit is out of scope; for that level of threat use a sacrificial machine.
- **Second layer on macOS.** QEMU itself runs inside a `sandbox-exec` profile: it can't read your home directory (only its own session folder and the base image) and it can't connect to services on your Mac's loopback, which `--net` would otherwise expose to the guest at `10.0.2.2`. Disable with `DOKIMASIA_NO_SANDBOX=1`. On Linux this layer is not implemented yet; with `--net`, treat anything listening on `127.0.0.1` as reachable from the VM.
- **Secrets stay out.** Copying a local directory leaves out `.env*`, private keys, `.npmrc`, `.netrc`, `.aws`, `.ssh` and similar (`--keep-secrets` to override). Repos cloned from a URL are copied as they are.
- **The base image is checked.** `init` verifies the Debian cloud image against Debian's published SHA512SUMS (same HTTPS origin: it catches corruption and a tampered mirror object, not a compromised origin).
- **Network off** means no NIC at all. `--net` adds QEMU user-mode NAT: code inside can then reach the internet (but not your files).
- **The host only clones.** `git clone` runs with hooks, submodules, fsmonitor, system and global git config disabled. Local directories are archived without following symlinks.
- **Output is untrusted text.** Terminal escape sequences are stripped before printing; nothing from the VM is ever executed on the host.
- If the `dokimasia` process itself is killed mid-`run`, the VM keeps running — `dokimasia ls` shows it, `dokimasia down --name …` removes it.
- Tested on **macOS / Apple Silicon**. Linux x86_64 / arm64 and WSL2 should work but are not verified yet — reports welcome.

## 🆚 Compared to

| | What runs in the VM | Your code | Agent |
|---|---|---|---|
| **dokimasia** | the untrusted repo | copied in, no mounts, no network | stays on the host, drives the VM |
| [clawk](https://github.com/clawkwork/clawk), [coop](https://github.com/trailofbits/coop) | the coding agent | mounted from the host, ssh-agent forwarded | inside the VM |
| [fend](https://github.com/Lisovate/fend) | `npm` commands | project dir shared via VirtioFS | — |
| Docker / firejail / bubblewrap | a process | bind mounts | shares the host kernel |

## 🤝 Contributing

Issues and PRs welcome. Everything lives in [`bin/dokimasia`](bin/dokimasia), and the agent rules are in [`skills/dokimasia/SKILL.md`](skills/dokimasia/SKILL.md).

## 📄 License

[MIT](LICENSE)
