# nex-profiles

Composable profile fragments for [nex](https://nex.styrene.io). Mix and match to build your system config.

## Quick start

```sh
# Install nex
curl -fsSL https://nex.styrene.io/install.sh | sh

# Bootstrap (installs Nix, Homebrew, Xcode CLT, scaffolds config)
nex init

# Apply a profile
nex profile apply your-user/your-profile

# Activate
nex switch
```

`nex init` handles the full dependency chain: Nix, Homebrew (which triggers Xcode CLT / git), config scaffolding, and first build. If git is missing after Homebrew, nex fails with a clear message pointing to `xcode-select --install`.

## Two ways to build a profile

### Option 1: Flat profile

A single `profile.toml` with everything inline. Simple, self-contained.

```toml
[meta]
name = "my-workstation"

[packages]
nix = ["git", "vim", "ripgrep"]

[shell.aliases]
ls = "eza"

[shell.env]
EDITOR = "nvim"
```

### Option 2: Compose from fragments

Pick fragments from this repo and compose them. Fragments are merged in order; later entries win on conflicts. Your profile's own inline sections are applied last as overrides.

```toml
[meta]
name = "my-mac"
compose = [
  "core/essentials",
  "platform/macos",
  "shell/bash",
  "role/dev-macos",
]

# Overrides applied after all fragments
[shell.aliases]
ls = "eza --icons --grid --group-directories-first"

[git]
name = "Your Name"
email = "you@example.com"
```

## Available fragments

| Category | Fragments | Description |
|----------|-----------|-------------|
| **core** | `essentials` | CLI tools, GNU userland, git defaults |
| **platform** | `macos`, `linux` | OS-level defaults (system prefs, nix settings) |
| **desktop** | `cosmic`, `gnome`, `kde`, `hyprland` | Desktop environment (Linux only) |
| **gpu** | `amd`, `nvidia`, `intel` | GPU drivers, Vulkan, hardware acceleration (Linux only) |
| **audio** | `pipewire`, `pulseaudio` | Audio backend (Linux only) |
| **shell** | `bash`, `zsh`, `fish` | Default shell, completions, history, PATH setup |
| **role** | `dev-macos`, `dev`, `gaming`, `server`, `edge` | Use-case packages and services; `dev-macos` is the macOS workstation role, `dev` is Linux-only |

## Merge rules

Fragments and overlays are merged in order. The rules:

| Data type | Rule | Example |
|-----------|------|---------|
| Aliases, env vars | Later wins on key conflict, others preserved | Overlay's `ls` overrides fragment's `ls` |
| Package lists | Union with dedup | Both fragments' packages included |
| Scalars (git name, macOS prefs) | Last writer wins | Overlay's git name replaces base's |
| `profileExtra` / `initExtra` | Appended if new content, deduplicated if same | Homebrew PATH from bash fragment preserved, not doubled |
| History settings | Last writer wins | Overlay's HISTSIZE overrides fragment's |

## Inheritance with `extends`

Profiles can extend other profiles across repos. The parent is applied first, then the child's sections override.

```
styrene-lab/nex-profiles      (public fragments and mac-dev base)
    │
    │  extends
    ▼
your-user/private-profile      (private: SSH keys, kubeconfig, work aliases)
```

`extends` and `compose` work together. If a profile has both, resolution order is:

1. The extended parent (recursively resolved)
2. Each compose fragment in order
3. The profile's own inline sections (overrides)

## Shell config pipeline

The `[shell]` section generates a home-manager nix module that manages bash dotfiles declaratively:

```
profile.toml [shell]
        │
        │  nex profile apply
        ▼
~/macos-nix/nix/modules/home/shell.nix    (generated nix module)
        │
        │  nex switch (darwin-rebuild)
        ▼
~/.profile      ← sessionVariables + profileExtra
~/.bash_profile ← sources ~/.profile, then ~/.bashrc
~/.bashrc       ← aliases, historySize, initExtra, bash-completion
```

### TOML fields

| Field | Maps to | Purpose |
|-------|---------|---------|
| `[shell.aliases]` | `programs.bash.shellAliases` | Shell aliases |
| `[shell.env]` | `home.sessionVariables` | Env vars exported at login |
| `profileExtra` | `programs.bash.profileExtra` | Login shell init (Homebrew PATH, Cargo, etc.) |
| `initExtra` | `programs.bash.initExtra` | Interactive shell init (pipefail, tool PATH, etc.) |

### History settings

`HISTSIZE`, `HISTFILESIZE`, and `HISTCONTROL` in `[shell.env]` are intercepted and mapped to native home-manager options (`programs.bash.historySize`, etc.). This is because home-manager sets its own history defaults in `.bashrc` that would override values set via `.profile` session variables.

## Git identity inheritance

**Important:** The `[git]` section sets `git config --global`. If you extend someone else's profile, their git identity is applied first. Override `[git]` in your own profile or you'll commit as the base profile's author.

```toml
# Always include in your profile
[git]
name = "Your Name"
email = "your-email@example.com"
```

## Keeping secrets out

Public profiles contain non-sensitive config only. Private config (kubeconfig paths, work aliases) goes in a private overlay:

```toml
# Private overlay (your-user/your-private-profile)
[meta]
extends = "styrene-lab/nex-profiles"

[shell.aliases]
clod = "claude --dangerously-skip-permissions"

[shell.env]
KUBECONFIG = "$HOME/.kube/work.kubeconfig"

[git]
name = "Your Name"
email = "your-email@example.com"
```

## Not yet implemented

These features are defined in fragment metadata but not yet enforced:

- **`requires`** — Fragments declare dependencies (e.g., `desktop/cosmic` requires `platform/linux`). The metadata exists in the TOML files but nex does not validate the dependency graph yet. Composing a Linux-only fragment on macOS won't error — it will just produce config that doesn't apply.
- **`[ssh]`** — SSH host configuration (`[ssh]`, `[[ssh.hosts]]` sections) is parsed without error but silently ignored. SSH config must be managed separately (e.g., via home-manager's `programs.ssh` in your nix config, or manual `~/.ssh/config`).
- **`nex profile init`** — Interactive builder that walks you through fragment selection. Not yet available; create your `profile.toml` manually for now.

## Making your own

1. Create a repo with a `profile.toml`
2. Either write a flat profile or use `compose` to pick fragments from this repo
3. Apply it: `nex profile apply your-user/your-profile`

See [nex.styrene.io](https://nex.styrene.io) for docs.
