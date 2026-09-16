# Podman Quadlets Configuration

This repository contains my personal, rootless Podman Quadlet configurations managed as Infrastructure as Code (IaC). It allows for identical, reproducible container deployments across multiple machines using GNU Stow and systemd.

## Prerequisites

Install `podman` and `stow` using your preferred package manager before proceeding.

**Headless systems:** rootless `systemd --user` services normally stop when you log out, and won't start at boot without an active login session. If you're running on a headless machine (no interactive login), enable lingering for your user so the services start at boot and keep running independent of any session:
```bash
loginctl enable-linger "$USER"
```

## Infrastructure Architecture

Each service lives in its own top-level directory, acting as an independent GNU Stow package. Every package mirrors the expected user config structure below it, allowing Stow to symlink its configuration files directly into place.

```text
podman-quadlets/
├── bambuddy
│   └── .config/containers/systemd/bambuddy.container
├── open-webui
│   └── .config/containers/systemd/open-webui.container
└── searxng
    └── .config/containers/systemd/searxng.container
```

This modular layout lets you stow (or unstow) individual services independently, so a machine can run just the containers it needs.

**Data Volumes:** Named Podman volumes (`open-webui`, `searxng_data`) and user-space bind mounts (`~/bambuddy`) are automatically initialized by Podman on first boot—no manual creation required.

**Networking:** Uses native host networking flags where appropriate for seamless local service interaction and device discovery.

**Updates:** Containers are configured with `AutoUpdate=registry` to integrate with systemd auto-update timers.

## Deployment with GNU Stow

To deploy these rootless Quadlets onto a fresh machine, execute the following steps:

1. Clone the repository to your central projects or dotfiles directory, then move into it:
   ```bash
   git clone git@github.com:m-fe02/podman-quadlets.git ~/Projects/podman-quadlets
   cd ~/Projects/podman-quadlets
   ```

2. **Symlink the configuration** into your home directory, naming the packages (services) you want on this machine:
   ```bash
   stow --no-folding -t ~ bambuddy open-webui searxng
   ```
   Omit any package you don't need on a given machine. `--no-folding` keeps Stow from collapsing the shared `.config/containers/systemd` directory into a single symlink, which would conflict once you stow a second package into it.

3. **Reload systemd** to pick up the newly linked Quadlet files. Quadlet generates transient service units from the `.container` files, so `systemctl --user enable` does not apply to them—`daemon-reload` alone is enough for the generator to pick up the new units and start them per their `[Install]` section:
   ```bash
   systemctl --user daemon-reload
   ```

4. Confirm the services came up:
   ```bash
   systemctl --user status open-webui.service searxng.service bambuddy.service
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

1. Stop the active container services. Since these are transient, generated units rather than static unit files, there is no persistent enablement to `disable`—removing the source `.container` file (next step) is what prevents them from being regenerated:
   ```bash
   systemctl --user stop open-webui.service searxng.service bambuddy.service
   ```

2. Remove the symlinks from your home directory using Stow's delete flag, naming the same packages you stowed (run from `~/Projects/podman-quadlets`):
   ```bash
   stow --no-folding -t ~ -D bambuddy open-webui searxng
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