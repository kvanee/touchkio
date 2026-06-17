# HBH fork of TouchKio — system updates + audio/display control

This is a fork of [leukipp/touchkio](https://github.com/leukipp/touchkio) for the
HBH Touch Panel. It adds HA entities (all on the *same* TouchKio MQTT device, so
there's no separate updater/control device) for: **system updates**, **LVA
mic/speaker selection**, **display power + rotation**, and an **assist debug
card** toggle.

Upstream TouchKio already shipped an **App** `update` entity and a read‑only
**Package Upgrades** sensor.

## What this fork adds

### System update entities (`update`)

| Entity | Install action | Support condition |
| --- | --- | --- |
| **OS Packages** (`<node>_os_update`) | `apt-get update && apt-get -y --with-new-pkgs upgrade`; auto‑reboots 10s after a successful install **if** a kernel/firmware change requires it | password‑less sudo |
| **Voice Assistant** (`<node>_lva_update`) | `git` fetch/checkout the tracked ref of `/opt/lva` → re‑run `script/setup` → `systemctl --user restart lva.service` | `git` + `/opt/lva` present |

### LVA audio device selection (`select`)

| Entity | Behavior | Support |
| --- | --- | --- |
| **Microphone** (`<node>_audio_input`) | options enumerated from LVA `--list-input-devices`; on change writes `~/.config/hbh/lva-input` and restarts `lva.service` | `/opt/lva` present |
| **Speaker** (`<node>_audio_output`) | options from LVA `--list-output-devices`; writes `~/.config/hbh/lva-output` + restart | `/opt/lva` present |

`lva-run.sh` reads those files and passes them to LVA's `--audio-input-device` /
`--audio-output-device` (falling back to XVF3800 auto-detect if unset).

### Display control

| Entity | Behavior | Support |
| --- | --- | --- |
| **Display Power** (`<node>_display_power`, `switch`) | `wlr-randr --output <out> --on/--off` | `wlr-randr` present |
| **Display Rotation** (`<node>_display_rotation`, `select`: normal/90/180/270) | `wlr-randr --transform`; persists `~/.config/hbh/rotation` so the labwc autostart re-applies it after reboot | `wlr-randr` present |

### Assist debug card (`switch`)

| Entity | Behavior | Support |
| --- | --- | --- |
| **Assist Debug Card** (`<node>_assist_debug_card`, `switch`, `entity_category: config`) | soft on/off (no hardware); persists `~/.config/hbh/assist-debug-card`. Read by the `assist-satellite-card` (`show_card_entity`) to show/hide its on-screen debug controls — the waveform overlay always runs regardless | always (disable via `--app-disable mqtt_assist_debug_card`) |

> TouchKio and LVA both run as **user** services in the same session, so audio/
> display commands and the LVA restart use the session tools + `systemctl --user`
> (no sudo), and `/opt/lva` is owned by the panel user — which is what lets
> TouchKio self-update and restart it.

The stock **Package Upgrades** sensor is kept; **OS Packages** is the actionable version.

### Files changed

- **`js/hardware.js`**
  - Updates: `installPackageUpgrades`, `checkLvaUpdate`, `installLvaUpdate`.
  - Audio: `getAudioInputDevices`/`getAudioOutputDevices` (via LVA `--list-*-devices`),
    `getSelectedAudioInput`/`Output`, `setSelectedAudioInput`/`Output` (persist + restart LVA).
  - Display: `getDisplayPower`/`setDisplayPower`, `getDisplayRotation`/`setDisplayRotation`
    (via `wlr-randr`), `displayOutputName`, `DISPLAY_ROTATIONS`.
  - `dirExists`/`ensureHbhDir`/`readValueFile` helpers; `checkSupport()` gains `lvaUpdate`,
    `audioSelect`, `displayControl` flags. New functions in `module.exports`.
- **`js/integration.js`**
  - `init/update` pairs: `OsUpdate`, `LvaUpdate`, `AudioInput`, `AudioOutput`, `DisplayPower`,
    `DisplayRotation`. Registered in `init()`; `DisplayPower` refreshed in the 1‑min `update()`.

No upstream lines are removed — additive, so it stays easy to rebase.

### Configuration (environment)

The fork reads these from the TouchKio process environment (set by the panel
installer's labwc autostart; defaults shown):

| Var | Default | Meaning |
| --- | --- | --- |
| `LVA_DIR` | `/opt/lva` | linux‑voice‑assistant checkout |
| `LVA_GIT_REF` | `main` | branch or tag to track (from `panel.env`) |
| `LVA_SERVICE` | `lva.service` | systemd unit to restart after an LVA update |

Each entity can be disabled like other TouchKio entities via `app_disable`:
`mqtt_os_update`, `mqtt_lva_update`, `mqtt_audio_input`, `mqtt_audio_output`,
`mqtt_display_power`, `mqtt_display_rotation`.

> **Ownership requirement:** because TouchKio runs as the panel user and performs
> the LVA `git` update + `systemctl --user restart` itself, `LVA_DIR` must be owned
> by that user and `lva.service` must be a user unit. The installer
> (`provisioning/boot-payload/hbh/install.sh`) sets this up.

## Update-entity state shape

Both entities publish a JSON payload to their `…/version/state` topic with the
keys Home Assistant's MQTT `update` platform parses natively (`installed_version`,
`latest_version`, `title`, `release_summary`, `release_url`, `update_percentage`,
`in_progress`). For **OS Packages**, `installed_version` is `"up to date"` and
`latest_version` becomes `"<N> updates"` when packages are pending, so HA shows an
update until `apt upgrade` clears the count.

## Building the `.deb`

Unchanged from upstream — the fork is pure JS:

```bash
yarn install
yarn make            # produces dist/touchkio_<ver>_arm64.deb (build on arm64 / Pi)
```

Point `panel.env`'s `TOUCHKIO_DEB_URL` at your fork's release asset so the panel
installer pulls this build.

## Verifying

After provisioning, the TouchKio device in Home Assistant shows three update
entities: **App** (TouchKio), **OS Packages**, and **Voice Assistant**. Clicking
*Install* runs the matching action on the Pi and republishes the new state.
Watch it with `journalctl --user -u touchkio -f` (or the labwc session log).
