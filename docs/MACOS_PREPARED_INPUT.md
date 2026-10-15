# Mac prepared package

[English](MACOS_PREPARED_INPUT.md) | [简体中文](MACOS_PREPARED_INPUT.zh-CN.md)

In the `prepared_input` workflow parameter, set the package’s HTTPS `url` and `sha256` to this package. The ZIP has this layout:

```text
manifest.json
product/
msl/b-<shader-chain>/
romfs_overlay/
resources/data/
  build_manifest.psv
  .build.config
generated/
  dm_vault_tab.h
  dm_hud_exp.h
  dm_bright_levels.inc
shader-chain.lock
c1_full.tsv
```

| Path | Contents |
|---|---|
| `manifest.json` | Format `d2r-mobile-macos-prepared`, schema `1`, source commit, game program hashes, generation key, producer toolchain and every file’s SHA-256 |
| `product/` | Upstream’s rewritten `touch` game programs and loader tables |
| `msl/` | macOS Metal libraries targeting macOS 15.0, and their reflection metadata; shared by both Mac apps |
| `romfs_overlay/` | Mac interface resources, including keyboard glyphs |
| `resources/data/` | Small metadata files needed by the Mac resource picker; used alongside existing Data |
| `generated/` | Data-table headers generated from game resources, used to compile the upstream macOS app's interface and runtime code |
| `shader-chain.lock`, `c1_full.tsv` | Matching shader tool fingerprints and index metadata |

Changes to the upstream macOS app's C/Objective-C interface, input handling, or runtime code can reuse this package when generation dependencies are unchanged. The build compares game programs and generation dependencies with the selected source and verifies every file before compiling.

## Prepare your own input

In a private repository, add a `GAME_ARCHIVE` Secret with a JSON object containing the HTTPS `url` and `sha256` of an archive with one `ExeFS`/`RomFS` pair. Use this workflow to generate the Mac prepared package:

```yaml
name: Prepare Mac input
on: workflow_dispatch
permissions:
  contents: write
  actions: write
jobs:
  prepare:
    uses: jiayx/d2r-mobile-build/.github/workflows/prepare-macos.yml@main
    secrets:
      GAME_ARCHIVE: ${{ secrets.GAME_ARCHIVE }}
```

The workflow restores the original archive cache when available, prepares Mac programs, interface resources and shaders, and publishes `D2RMobile-MacPrepared.zip` and `MacPrepared-SHA256SUMS.txt` to your private Releases. Use this ZIP as the input to **Build macOS**. An asset URL in the same private repository can use `https://api.github.com/repos/OWNER/REPO/releases/assets/ASSET_ID`; downloads authenticate with that repository’s workflow token.
