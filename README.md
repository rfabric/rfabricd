# rfabricd

The rFabric daemon: the Rust process that runs on every [rFabric](https://rfabric.io)-managed robot. It bridges the robot to the rFabric platform: provisioning and identity, telemetry and commands, bulk data uploads, deployments, evaluation runs, and interactive media and teleoperation.

It ships as two builds of the same program. A robot installs one:

| Build | What it is | Runtime dependencies |
| --- | --- | --- |
| `rfabricd` (default) | Everything, including interactive media (live video) and teleoperation | GStreamer |
| `rfabricd-core` | The same subcommands, config and provisioning, without interactive media and teleoperation (no GStreamer, no LiveKit) | none |

- Linux and macOS on amd64 and arm64
- systemd-managed on Debian/Ubuntu and RHEL/Fedora via the official `.deb` / `.rpm` packages
- X.509 device identity minted on first boot from a one-time bootstrap token

## Install

```bash
# Linux / macOS - on Debian/Ubuntu (apt) and Fedora/RHEL (dnf) installs the rfabricd package
curl -fsSL https://get.rfabric.io/rfabricd.sh | sh

# The minimal build rfabricd-core instead (either form)
curl -fsSL https://get.rfabric.io/rfabricd.sh | sh -s -- --core
curl -fsSL https://get.rfabric.io/rfabricd.sh | RFABRICD_BUILD=core sh

# macOS / Linux (Homebrew) - dev laptops, custom images
brew install rfabric/tap/rfabricd          # or rfabric/tap/rfabricd-core

# Binary only (no package, no systemd service) - bare-metal installs, CI runners
curl -fsSL https://get.rfabric.io/rfabricd.sh | RFABRIC_INSTALL_DIR=/usr/local/bin sh
```

On Linux with `apt-get` or `dnf`, `rfabricd.sh` adds the rFabric package repository <https://linux.rfabric.io> (using `sudo` when not root), installs the `rfabricd` package and enables `rfabricd.service`. The package ships the binary at `/usr/bin/rfabricd`, the systemd unit at `/lib/systemd/system/rfabricd.service`, the `rfabric` system user, and `/etc/rfabric/rfabricd.example.toml` as a starting point. The `rfabricd-core` package has the same shape with `/usr/bin/rfabricd-core` and `rfabricd-core.service`.

On macOS and other Linux systems, or with `RFABRIC_VERSION` / `RFABRIC_INSTALL_DIR` set, the installer places the checksum-verified binary from the latest published GitHub Release into `RFABRIC_INSTALL_DIR` (default `/usr/local/bin`). When that release has no `rfabricd` build for your platform, it stops with an error naming the platform and the release; it never installs `rfabricd-core` in its place. Homebrew and the binary-only install only place the binary on `PATH` - operators wanting a managed service use the package.

Upgrade with `sudo apt-get upgrade` / `sudo dnf upgrade`.

Release candidates: `curl -fsSL https://get.rfabric.io/rfabricd.sh | RFABRIC_CHANNEL=rc sh`. On apt/dnf systems this switches the package repository to the `rc` channel; rerunning `rfabricd.sh` without `RFABRIC_CHANNEL` switches back to `stable`.

## Quick start

The commands are the same for both builds; with `rfabricd-core`, run `rfabricd-core` and use `rfabricd-core.service`.

### Dev laptop / Homebrew - no sudo, no manual config copy

```bash
rfabricd provision prov_...
rfabricd run
```

`provision` writes the cert / key / CA triple, `provisioned.toml`, and a freshly generated `rfabricd.toml` into `~/.config/rfabric/` (override with `--out-dir`). `rfabricd run` picks the same directory up automatically. Re-running `provision` refreshes the certificate but leaves an existing `rfabricd.toml` untouched, so operator edits survive.

There is no shared service token to copy: rfabricd authenticates to the rFabric API by exchanging its X.509 client certificate for a short-lived `type=robot` JWT against the mTLS robot-auth endpoint baked in at build time (override with `--robot-auth-url` on `provision` for staging / on-prem).

The bootstrap token can also be supplied via `RFABRICD_PROVISIONING_TOKEN` to keep it out of shell history. The API endpoint is baked in at build time from the release channel (`stable` -> prod, otherwise -> dev); pass `--api-url https://api.<env>.rfabric.io` only when targeting a non-default environment.

### Production robot (apt / dnf) - systemd-managed

```bash
sudo rfabricd provision prov_... --out-dir /etc/rfabric

sudo systemctl enable --now rfabricd
sudo journalctl -u rfabricd -f
```

The systemd unit pins `RFABRICD_CONFIG=/etc/rfabric/rfabricd.toml` and reads overrides from `/etc/rfabric/rfabricd.env`, so the service always reads the operator-owned config regardless of the per-user default.

If you forget `--out-dir` while running under `sudo`, `provision` prints a warning and falls back to `/root/.config/rfabric/`, a path the systemd unit does **not** read. Always pass `--out-dir /etc/rfabric` for the system install (or re-run without sudo for the per-user install).

`rfabricd --help` and `rfabricd <subcommand> --help` document every flag. `rfabricd version --output json` reports build channel, variant (`media` for `rfabricd`, `core` for `rfabricd-core`), commit, and host triple, the same shape as `rfabric version --output json`.

## Documentation

- Operator guide: <https://docs.rfabric.io/rfabricd>
- Platform docs: <https://docs.rfabric.io>

## Issues & support

Report bugs and request features in this repo's [issue tracker](https://github.com/rfabric/agent/issues).
