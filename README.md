# Podman Quadlets Configuration

This repository contains my personal, rootless Podman Quadlet configurations managed as Infrastructure as Code (IaC). It allows for identical, reproducible container deployments across multiple machines using GNU Stow and systemd.

## Infrastructure Architecture

The repository mirrors the expected user config structure, allowing GNU Stow to symlink configuration files directly into place.

```text
podman-quadlets/
└── .config
    └── containers
        └── systemd
            ├── bambuddy.container
            ├── open-webui.container
            └── searxng.container
```

**Data Volumes:** Named Podman volumes (`open-webui`, `searxng_data`) and user-space bind mounts (`~/bambuddy`) are automatically initialized by Podman on first boot—no manual creation required.

**Networking:** Uses native host networking flags where appropriate for seamless local service interaction and device discovery.

**Updates:** Containers are configured with `AutoUpdate=registry` to integrate with systemd auto-update timers.

## Deployment with GNU Stow

To deploy these rootless Quadlets onto a fresh machine, execute the following steps:

1. Clone the repository to your central projects or dotfiles directory:
   ```bash
   git clone git@github.com:m-fe02/podman-quadlets.git ~/Projects/podman-quadlets
   ```

2. **Symlink the configuration** into your home directory using explicit targeting:
   ```bash
   stow -d ~/Projects -t ~ podman-quadlets
   ```

3. Force systemd to parse the newly linked Quadlet files and generate transient service units:
   ```bash
   systemctl --user daemon-reload
   ```

4. **Enable and start** the generated container services simultaneously:
   ```bash
   systemctl --user enable --now open-webui.service searxng.service bambuddy.service
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

## Decommission / Unstow

If you need to tear down the environment, pull back the symlinks, or clean up the host paths, follow this order to prevent orphaned systemd services:

1. Stop and disable the active container services:
   ```bash
   systemctl --user disable --now open-webui.service searxng.service bambuddy.service
   ```

2. Remove the symlinks from your home directory using Stow's delete flag:
   ```bash
   stow -d ~/Projects -t ~ -D podman-quadlets
   ```

3. **Reload systemd** to clear out the generated transient service definitions:
   ```bash
   systemctl --user daemon-reload
   ```
