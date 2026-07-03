# quadlet-rustdesk

Quadlet setup for [RustDesk](https://rustdesk.com) — a self-hosted remote desktop server (`docker.io/rustdesk/rustdesk-server:latest`).

This project was created with the help of Claude Code and https://github.com/mkoester/quadlet-my-guidelines/blob/main/new_quadlet_with_ai_assistance.md.

## Architecture

RustDesk uses two server components:

| Container | Binary | Role |
|---|---|---|
| `rustdesk-hbbs` | `hbbs` | Rendezvous server — handles ID registration, NAT traversal, and peer discovery |
| `rustdesk-hbbr` | `hbbr` | Relay server — forwards traffic when a direct P2P connection cannot be established |

Both containers share the same data directory (`~rustdesk/data` → `/root` in the container) to share the server key pair (`id_ed25519` / `id_ed25519.pub`). Clients authenticate both servers using the same public key.

## Files in this repo

| File | Description |
|---|---|
| `rustdesk.network` | Podman network shared by hbbs and hbbr |
| `rustdesk-hbbs.container` | Quadlet unit for the rendezvous server |
| `rustdesk-hbbr.container` | Quadlet unit for the relay server |
| `rustdesk.env` | Default environment variables |
| `rustdesk.override.env.template` | Template for local overrides (relay hostname) |
| `rustdesk-backup.service` | Systemd service: backs up the data directory |
| `rustdesk-backup.timer` | Systemd timer: triggers the backup daily |

## Setup

```sh
# 1. Create service user (regular user, home in /var/lib)
sudo useradd -m -d /var/lib/rustdesk -s /usr/sbin/nologin rustdesk

REPO_URL=https://github.com/mkoester/quadlet-rustdesk.git
REPO=~rustdesk/quadlet-rustdesk
```

```sh
# 2. Enable linger
sudo loginctl enable-linger rustdesk

# 3. Clone this repo into the service user's home
sudo -u rustdesk git clone $REPO_URL $REPO

# 4. Create quadlet and data directories
sudo -u rustdesk mkdir -p ~rustdesk/.config/containers/systemd
sudo -u rustdesk mkdir -p ~rustdesk/data

# 5. Create .override.env from template and fill in required values
sudo -u rustdesk cp $REPO/rustdesk.override.env.template $REPO/rustdesk.override.env
sudo -u rustdesk nano $REPO/rustdesk.override.env

# 6. Symlink all quadlet files from the repo
sudo -u rustdesk ln -s $REPO/rustdesk.network ~rustdesk/.config/containers/systemd/rustdesk.network
sudo -u rustdesk ln -s $REPO/rustdesk-hbbs.container ~rustdesk/.config/containers/systemd/rustdesk-hbbs.container
sudo -u rustdesk ln -s $REPO/rustdesk-hbbr.container ~rustdesk/.config/containers/systemd/rustdesk-hbbr.container
sudo -u rustdesk ln -s $REPO/rustdesk.env ~rustdesk/.config/containers/systemd/rustdesk.env
sudo -u rustdesk ln -s $REPO/rustdesk.override.env ~rustdesk/.config/containers/systemd/rustdesk.override.env

# 7. Reload and start
sudo -u rustdesk XDG_RUNTIME_DIR=/run/user/$(id -u rustdesk) systemctl --user daemon-reload
sudo -u rustdesk XDG_RUNTIME_DIR=/run/user/$(id -u rustdesk) systemctl --user start rustdesk-hbbr rustdesk-hbbs

# 8. Verify
sudo -u rustdesk XDG_RUNTIME_DIR=/run/user/$(id -u rustdesk) systemctl --user status rustdesk-hbbr rustdesk-hbbs
```

## Configuration

`rustdesk.env` contains non-sensitive defaults:

| Variable | Default | Description |
|---|---|---|
| `ENCRYPTED_ONLY` | `0` | Set to `1` to reject unencrypted connections |

`rustdesk.override.env` (created from the template) holds instance-specific values:

| Variable | Description |
|---|---|
| `RELAY` | Public hostname or IP of this server, used by clients to reach hbbr (port 21117). Must be reachable from the internet. |

## Ports

RustDesk communicates directly over its own ports — it is **not** behind a reverse proxy. All ports are published on all interfaces.

| Port | Protocol | Service | Purpose |
|---|---|---|---|
| 21115 | TCP | hbbs | NAT type test |
| 21116 | TCP | hbbs | ID registration and heartbeat |
| 21116 | UDP | hbbs | Hole punching |
| 21117 | TCP | hbbr | Relay traffic |
| 21118 | TCP | hbbs | WebSocket (web client) |
| 21119 | TCP | hbbr | WebSocket relay (web client) |

### Firewall (firewalld)

All six ports must be opened to allow inbound connections from RustDesk clients:

```sh
sudo firewall-cmd --permanent --add-port=21115/tcp
sudo firewall-cmd --permanent --add-port=21116/tcp
sudo firewall-cmd --permanent --add-port=21116/udp
sudo firewall-cmd --permanent --add-port=21117/tcp
sudo firewall-cmd --permanent --add-port=21118/tcp
sudo firewall-cmd --permanent --add-port=21119/tcp
sudo firewall-cmd --reload
```

Works similarly with other firewall solutions.

## Client configuration

After setup, retrieve the public key from the data directory:

```sh
sudo cat ~rustdesk/data/id_ed25519.pub
```

In the RustDesk client, go to **Settings → Network** and set:
- **ID server**: `<your-server-hostname>`
- **Key**: contents of `id_ed25519.pub`

## Backup

The backup copies the data directory (server key pair + peer database) to `/var/backups/rustdesk/`. Requires `rsync` to be installed on the host (`sudo dnf install rsync` / `sudo apt install rsync`). A remote machine pulls via `rsync` over SSH using the shared `backupuser`. See the [general backup setup](https://github.com/mkoester/quadlet-my-guidelines#backup) for the one-time server-wide setup (group, backup user, SSH key).

```sh
# 1. Create backup staging directory (owned by rustdesk, readable by backup-readers group)
sudo mkdir -p /var/backups/rustdesk
sudo chown rustdesk:backup-readers /var/backups/rustdesk
sudo chmod 2750 /var/backups/rustdesk

# 2. Symlink the backup service and timer from the repo
sudo -u rustdesk mkdir -p ~rustdesk/.config/systemd/user
sudo -u rustdesk ln -s $REPO/rustdesk-backup.service ~rustdesk/.config/systemd/user/rustdesk-backup.service
sudo -u rustdesk ln -s $REPO/rustdesk-backup.timer ~rustdesk/.config/systemd/user/rustdesk-backup.timer

# 3. Enable and start the timer
sudo -u rustdesk XDG_RUNTIME_DIR=/run/user/$(id -u rustdesk) systemctl --user daemon-reload
sudo -u rustdesk XDG_RUNTIME_DIR=/run/user/$(id -u rustdesk) systemctl --user enable --now rustdesk-backup.timer
```

### On the remote (backup) machine

```sh
rsync -az backupuser@rustdesk-host:/var/backups/rustdesk/ /path/to/local/backup/rustdesk/
```

## Notes

- The server key pair (`id_ed25519` / `id_ed25519.pub`) is generated on first start and stored in `~rustdesk/data/`. Back it up — if the key changes, all clients must be reconfigured.
- Both containers publish ports on all interfaces (no `127.0.0.1:` bind) — remember to configure your firewall (see above).
- Both containers run as root inside the container (UID 0), which maps to the `rustdesk` host user via rootless Podman's user namespace.
- The shared data volume uses `:z` (lowercase) so both hbbs and hbbr can mount it simultaneously under SELinux.
- No health check is configured: the image's built-in health check (`/usr/bin/healthcheck.sh`) is designed for the single-container s6-overlay variant, not the two-container model.
- `AutoUpdate=registry` is enabled; activate the timer once to get automatic image updates:
  ```sh
  sudo -u rustdesk XDG_RUNTIME_DIR=/run/user/$(id -u rustdesk) systemctl --user enable --now podman-auto-update.timer
  ```
- To prune old images automatically, enable the system-wide prune timer (see [image pruning setup](https://github.com/mkoester/quadlet-my-guidelines#image-pruning) for the one-time system setup). Replace `30` with the desired retention period in days:
  ```sh
  sudo -u rustdesk XDG_RUNTIME_DIR=/run/user/$(id -u rustdesk) systemctl --user enable --now podman-image-prune@30.timer
  ```
