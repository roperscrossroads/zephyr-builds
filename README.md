# zephyr-builds

Device-agnostic build harness for [Zephyr RTOS](https://www.zephyrproject.org/)
firmware. One `west` manifest set + one CI matrix builds **many boards** against
**two version channels**, so a board's firmware can be built reproducibly
against a pinned Zephyr release *and* against bleeding-edge `main`.

## Channels

The stable/bleeding-edge split is a single knob: the `zephyr` project
`revision:` in the manifest. Each channel is a separate manifest file, and CI
picks between them per build.

| Channel | Manifest | Zephyr `revision:` | Moves how |
|---|---|---|---|
| `stable` | `west-stable.yml` | a release **tag** (`v4.4.1`) | by hand, when a release is validated |
| `main` | `west-main.yml` | a full **SHA** on `main` | bot-bumped nightly ([`bump-main.yml`](.github/workflows/bump-main.yml)) |

`main` is pinned to a SHA (not the moving branch) so every nightly build is
reproducible from git history. A broken `main` build never masks `stable`
(`fail-fast: false`).

## Boards (targets)

The build app is fitted to each board — a smoke build only means something if it
exercises what the board is for.

| Board (`west -b`) | App | Toolchain | Note |
|---|---|---|---|
| `rpi_4b` | `samples/hello_world` | `aarch64-zephyr-elf` | AArch64 SBC |
| `rpi_pico/rp2040/w` | `samples/hello_world` | `arm-zephyr-eabi` | RP2040 + CYW43 (Pico W) |
| `heltec_wifi_lora32_v3/esp32s3/procpu` | `samples/drivers/lora/send` | `xtensa-espressif_esp32s3_zephyr-elf` | ESP32-S3 + SX1262 (LoRa) |
| `xiao_ble` | `samples/hello_world` | `arm-zephyr-eabi` | nRF52840 — LoRa target later (SX1262 overlay) |

Matrix = `channel × target` (8 cells). Apps are upstream Zephyr's own samples,
fetched by `west update` — no app source lives in this repo yet.

## Layout

```
west-stable.yml                 # imports zephyr @ release tag  + module allowlist
west-main.yml                   # imports zephyr @ bot-bumped SHA + module allowlist
.github/workflows/
  ci.yml         # reusable: 4-board matrix build + per-channel release (input: channel)
  main.yml       # DAILY (+ dispatch, + push to west-main.yml)   → ci.yml(main)   → 'nightly'
  stable.yml     # WEEKLY (+ dispatch, + push to west-stable.yml) → ci.yml(stable) → 'stable'
  bump-main.yml  # 04:00 daily: rewrite west-main.yml SHA to upstream main tip, commit
```

The channels are split by **cadence**: `main` tracks upstream Zephyr so it builds
daily; `stable` is a pinned tag with byte-identical output, so it only rebuilds
on change, on demand, or a weekly reproducibility heartbeat.

## Build locally

Needs the Zephyr SDK the pinned release expects — `v4.4.1` requires **SDK 1.0.1**
(`west sdk install` fetches the right one). Mirror the CI workspace layout so the
sample paths resolve:

```sh
mkdir -p /tmp/zb/app && cp west-stable.yml /tmp/zb/app/west.yml
cd /tmp/zb && west init -l app && west update      # zephyr lands at /tmp/zb/zephyr
west sdk install -t aarch64-zephyr-elf             # once, if not already installed
west build -p always -b rpi_4b -d build zephyr/samples/hello_world
```

Swap `west-main.yml` for the bleeding-edge channel. There are no repo-local git
hooks — CI is the only build gate.

## Downloads

Each channel publishes its own release — every board's `zephyr.bin` + `zephyr.elf`
named `<channel>-<board>.{bin,elf}`, plus a `sha256sums.txt`. Assets are
regenerated each run, so the URLs are stable:

| Release | Channel | Contents | Refreshed |
|---|---|---|---|
| **`stable`** | v4.4.1 tag | `stable-<board>.{bin,elf}` | on change / weekly |
| **`nightly`** *(pre-release)* | Zephyr `main` | `main-<board>.{bin,elf}` | daily |

```
https://github.com/roperscrossroads/zephyr-builds/releases/download/stable/stable-heltec_v3.bin
```

Verify a download:

```sh
base=https://github.com/roperscrossroads/zephyr-builds/releases/download/stable
curl -LO "$base/sha256sums.txt"
curl -LO "$base/stable-heltec_v3.bin"
sha256sum -c sha256sums.txt --ignore-missing
```

A red board drops out of its release rather than blocking the others. Tagged,
permanent releases will come with real firmware.

## License

MIT — see [`LICENSE`](LICENSE).

This repo contains only build orchestration (west manifests + CI workflows). The
Zephyr RTOS and its modules are **fetched at build time** via `west` and are not
vendored here; they remain under their own licenses (Zephyr itself is Apache-2.0).
