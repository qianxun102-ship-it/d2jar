# Prepared package

[English](PREPARED_INPUT.md) | [简体中文](PREPARED_INPUT.zh-CN.md)

Supply an HTTPS link to a ZIP, tar.gz, or tar.xz package in the `prepared_input` workflow parameter. The required files are shown below; one enclosing directory is supported.

```text
prepared.json
source-payload.zip
shader-chain.lock
resource-manifest.json
generated/
  host/dm_vault_tab.h
  host/dm_hud_exp.h
  host/dm_bright_levels.inc
  ios/Sources/dm_touch_native_art.c
```

| File | Contents |
|---|---|
| `prepared.json` | Format `d2r-mobile-prepared`, schema `1`, game versions, program checksums, payload metadata and each file’s size and SHA-256. `producer.source_commit` identifies the source used to generate the package. |
| `source-payload.zip` | Generated game program, interface resources, overlays and binary iOS Metal libraries under `Payload/DmIOS.app/`. |
| `shader-chain.lock` | Shader toolchain fingerprints. |
| `resource-manifest.json` | Names, sizes and checksums of the 18 Data files for this game resource version, shared by iOS and macOS. |
| `generated/` | Data-table headers and a skill-atlas index C source file generated from game resources, used to compile the upstream iOS app's interface and runtime code. |

The build checks package integrity, game programs, resource metadata, and shader consistency. It compares generation dependencies with `producer.source_commit` in the selected source repository. Changes to the upstream iOS app's C/Objective-C interface, input handling, or runtime code can reuse the package when generation dependencies are unchanged; changes to generation rules require a matching updated package.

The workflow downloads this package and compiles the selected source into two IPAs. Download Data and the Legacy Device Pack separately from your resource provider, then follow the [installation instructions](../README.md#download-and-use-on-ios).

## Generate an iOS prepared package

In your private repository, configure two Secrets. Each contains a JSON object with an HTTPS download `url` and `sha256`:

| Secret | Input |
|---|---|
| `GAME_ARCHIVE` | An archive containing one `ExeFS` / `RomFS` pair; the same original input used for Mac preparation |
| `LEGACY_METADATA` | A ZIP containing `pack.json` and `pack_index.tsv` from the Legacy pack generation output for the same game resource version |

Save the following as `.github/workflows/prepare-ios-input.yml`, then run **Actions → Prepare iOS input → Run workflow**:

```yaml
name: Prepare iOS input
on:
  workflow_dispatch:
    inputs:
      source_repository:
        description: Source repository
        type: string
        default: jhaizhou-ops/d2r-mobile
      source_ref:
        description: Branch, tag or commit; empty selects the latest release
        type: string
        default: ''
permissions:
  contents: write
  actions: write
jobs:
  prepare:
    uses: jiayx/d2r-mobile-build/.github/workflows/prepare-ios.yml@main
    with:
      source_repository: ${{ inputs.source_repository }}
      source_ref: ${{ inputs.source_ref }}
    secrets:
      GAME_ARCHIVE: ${{ secrets.GAME_ARCHIVE }}
      LEGACY_METADATA: ${{ secrets.LEGACY_METADATA }}
      SOURCE_TOKEN: ${{ secrets.SOURCE_TOKEN }}
```

The workflow defaults to the latest official upstream Release. It generates the rewritten game programs, interface assets, overlays, host headers, and precompiled iOS Metal shaders. The existing Legacy index is used to produce older-device overlays, reusing the matching texture pack.

The original game archive and completed preparation outputs are cached by their inputs and generation dependencies. The workflow publishes `D2RMobile-Prepared.zip` and `Prepared-SHA256SUMS.txt` to your private Releases. Set the ZIP’s URL and checksum in your `prepared_input` JSON, then use **Build IPA** for ordinary builds.
