# Adv360-Pro-ZMK — Kinesis Advantage 360 Pro (ZMK)

## Source of truth

- **`config/adv360.keymap`** is what ZMK compiles/flashes. Edit this.
- `config/keymap.json` is STALE / drifted (older `&mt` mods, plain `&kp SPACE`). Do **not** regenerate from it — it would clobber the hand-maintained `&hm` homerow mods. It's only used by the web/keymap editors.
- `config/version.dtsi` is **generated** (git-ignored) by `bin/get_version.sh` — it defines the `macro_ver` label used in `layer_mod`. Never commit it.

## Layers

0 = default · 1 = Sys+Num · 2 = Brd+Sys (right-hand arrows: HOME/UP/END on U-I-O, LEFT/DOWN/RIGHT on J-K-L) · 3 = mod/BT · 4 = GameArrows (left-hand WASD arrows).

Left-thumb Space = `&lt 2 SPACE` (tap=Space, hold=arrows) — mirrors the cyboard's `LT1(SPACE)`.

## Deploy option A — GitHub Actions (normal)

Push to `origin` (branch `V3.0`) → `build.yml` runs → download the `firmware` artifact (`*-left.uf2`, `*-right.uf2`) → flash.
NOTE: pushing to `fugit/*` currently fails — local SSH keys auth as the work account `KeithOBrien-RGA`, which lacks access. See the fugit-vs-work SSH notes.

## Deploy option B — local Docker build (no push / offline-ish)

Bypasses CI and the push-key problem. `docker` here is the **podman** shim, so images must be **fully-qualified** (`docker.io/...`).

First-time (clones ZMK+Zephyr into the repo via `west update`, ~15-20 min):

```bash
cd /home/keith/repos/github.com/fugit/Adv360-Pro-ZMK
bin/get_version.sh   # REQUIRED — generates config/version.dtsi (defines macro_ver)
docker run --rm -v "$PWD":/workspace -w /workspace \
  docker.io/zmkfirmware/zmk-build-arm:stable bash -c '
    west init -l config && west update && west zephyr-export &&
    west build -p -s zmk/app -d build/left  -b adv360_left  -- -DZMK_CONFIG=/workspace/config &&
    west build -p -s zmk/app -d build/right -b adv360_right -- -DZMK_CONFIG=/workspace/config &&
    cp build/left/zephyr/zmk.uf2  firmware/local-left.uf2 &&
    cp build/right/zephyr/zmk.uf2 firmware/local-right.uf2'
```

Rebuild (workspace already cloned) — DROP `west init`/`west update` (they error "already in a west workspace"), but KEEP `west zephyr-export` (it registers Zephyr's CMake package in the container's ephemeral HOME, lost each `--rm` run):

```bash
bin/get_version.sh
docker run --rm -v "$PWD":/workspace -w /workspace \
  docker.io/zmkfirmware/zmk-build-arm:stable bash -c '
    west zephyr-export &&
    west build -p -s zmk/app -d build/left  -b adv360_left  -- -DZMK_CONFIG=/workspace/config &&
    west build -p -s zmk/app -d build/right -b adv360_right -- -DZMK_CONFIG=/workspace/config &&
    cp build/left/zephyr/zmk.uf2  firmware/local-left.uf2 &&
    cp build/right/zephyr/zmk.uf2 firmware/local-right.uf2'
```

### Gotchas hit (and fixed)
- **`gen_defines.py failed … undefined node label 'macro_ver'`** → forgot `bin/get_version.sh`; `version.dtsi` was empty.
- **`short-name … did not resolve`** → podman needs `docker.io/` prefix.
- **`Could not find … Zephyr` (ZephyrConfig.cmake)** → dropped `west zephyr-export`; it must run every container.
- **`docker run` hangs on pull/`west update`** → broken IPv6 to registry/GitHub; prefer IPv4 (`echo 'precedence ::ffff:0:0/96 100' | sudo tee -a /etc/gai.conf`).

## Flashing

Left = **Mod+macro1**, right = **Mod+macro3** → each mounts as a USB drive → copy the matching `.uf2`. Spurious "not ejected" error after copy is normal. Physical reset buttons also enter bootloader.
