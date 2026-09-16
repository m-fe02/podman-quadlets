# Podman Quadlets Configuration

This repository contains my personal, rootless Podman Quadlet configurations managed as Infrastructure as Code (IaC). It allows for identical, reproducible container deployments across multiple machines using GNU Stow and systemd.

## Prerequisites

Install `podman` and `stow` using your preferred package manager before proceeding.

## Infrastructure Architecture

Each service lives in its own top-level directory, acting as an independent GNU Stow package. Every package mirrors the expected user config structure below it, allowing Stow to symlink its configuration files directly into place.

```text
podman-quadlets/
├── bambuddy
│   └── .config/containers/systemd/bambuddy.container
├── ollama
│   └── .config/containers/systemd/ollama.container
├── open-webui
│   └── .config/containers/systemd/open-webui.container
└── searxng
    └── .config/containers/systemd/searxng.container
```

This modular layout lets you stow (or unstow) individual services independently, so a machine can run just the containers it needs.

**Data Volumes:** Named Podman volumes (`open-webui`, `searxng_data`) and user-space bind mounts (`~/bambuddy`, `~/.local/share/ollama`) are automatically initialized by Podman on first boot—no manual creation required.

**Networking:** Uses native host networking flags where appropriate for seamless local service interaction and device discovery. `ollama` additionally passes through AMD ROCm GPU devices for hardware-accelerated inference.

**Updates:** `bambuddy`, `open-webui`, and `searxng` are configured with `AutoUpdate=registry` to integrate with systemd auto-update timers. `ollama` is pinned manually since it tracks a specific ROCm-tagged image.

## Deployment with GNU Stow

To deploy these rootless Quadlets onto a fresh machine, execute the following steps:

1. Clone the repository to your central projects or dotfiles directory:
   ```bash
   git clone git@github.com:m-fe02/podman-quadlets.git ~/Projects/podman-quadlets
   ```

2. **Symlink the configuration** into your home directory, naming the packages (services) you want on this machine:
   ```bash
   stow -d ~/Projects/podman-quadlets -t ~ bambuddy ollama open-webui searxng
   ```
   Omit any package you don't need on a given machine—for example, skip `ollama` on a host without a compatible GPU.

3. Force systemd to parse the newly linked Quadlet files and generate transient service units:
   ```bash
   systemctl --user daemon-reload
   ```

4. **Enable and start** the generated container services simultaneously:
   ```bash
   systemctl --user enable --now open-webui.service searxng.service bambuddy.service ollama.service
   ```

## Keeping Containers Up to Date

Containers configured with `AutoUpdate=registry` can be updated via Podman's built-in auto-update mechanism.

Check which containers have newer images available without pulling:
```bash
podman auto-update --dry-run
```

Pull updated images and restart affected containers:
```bash
podman auto-update
```

You can clean up dangling images after updated images are pulled:

```bash
podman image prune -f
```

## Decommission / Unstow

If you need to tear down the environment, pull back the symlinks, or clean up the host paths, follow this order to prevent orphaned systemd services:

1. Stop and disable the active container services:
   ```bash
   systemctl --user disable --now open-webui.service searxng.service bambuddy.service ollama.service
   ```

2. Remove the symlinks from your home directory using Stow's delete flag, naming the same packages you stowed:
   ```bash
   stow -d ~/Projects/podman-quadlets -t ~ -D bambuddy ollama open-webui searxng
   ```

3. **Reload systemd** to clear out the generated transient service definitions:
   ```bash
   systemctl --user daemon-reload
   ```

4. Finally, you can remove the installed container images.

Check what is installed with:

```bash
podman images
```

Remove with:

```bash
podman image rm <image_hash>
```