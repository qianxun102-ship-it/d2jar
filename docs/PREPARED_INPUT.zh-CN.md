# 预处理包

[English](PREPARED_INPUT.md) | [简体中文](PREPARED_INPUT.zh-CN.md)

在运行参数 `prepared_input` 中填写 ZIP、tar.gz 或 tar.xz 预处理包的 HTTPS 链接。必需文件如下，允许外面再套一层文件夹。

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

| 文件 | 内容 |
|---|---|
| `prepared.json` | 格式标识 `d2r-mobile-prepared`、格式版本 `1`、游戏版本、程序校验值、payload 元数据及各文件大小和 SHA-256。`producer.source_commit` 标识生成此包时的源码提交。 |
| `source-payload.zip` | `Payload/DmIOS.app/` 下已生成的游戏程序、界面素材、覆盖层和二进制 iOS Metal 着色器。 |
| `shader-chain.lock` | 着色器工具链指纹。 |
| `resource-manifest.json` | 该游戏资源版本共用的 18 个 Data 文件的名称、大小和校验值；iOS 与 macOS 共用。 |
| `generated/` | 从游戏资源生成的数据表头文件和技能图集索引 C 源文件，供上游 iOS 应用的界面及运行时代码编译使用。 |

构建会检查包完整性、游戏程序、资源元数据和着色器的一致性，并在所选源码仓库中对照 `producer.source_commit` 检查生成依赖。只修改上游 iOS 应用的界面、输入处理或运行时等 C/Objective-C 代码，且不涉及生成依赖时，可复用此包；生成规则变化时需使用配套的新包。

工作流下载此包，并将所选源码编译成两份 IPA。Data 和 Legacy Device Pack 从资源提供者的网盘另行下载，导入方式见[安装说明](../README.zh-CN.md#ios-下载和使用)。

## 生成 iOS 预处理包

在自己的私有仓库配置两个 Secret，均使用包含 HTTPS 下载地址 `url` 和校验值 `sha256` 的 JSON：

| Secret | 输入 |
|---|---|
| `GAME_ARCHIVE` | 包含一对 `ExeFS` / `RomFS` 的原始游戏文件压缩包，与 Mac 预处理使用同一份输入 |
| `LEGACY_METADATA` | 包含 `pack.json` 和 `pack_index.tsv` 的 ZIP，两份文件取自同一游戏资源版本的 Legacy 包生成结果 |

将以下内容保存为 `.github/workflows/prepare-ios-input.yml`，然后在 **Actions → Prepare iOS input → Run workflow** 执行：

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

工作流默认使用上游最新正式 Release，生成改写后的游戏程序、界面素材、覆盖层、宿主表头和预编译 iOS Metal 着色器。Legacy 索引用于生成老设备覆盖层，沿用现有贴图包。

原始游戏压缩包和完成的预处理结果会按输入与生成规则缓存。结果发布到自己的私有 Releases，包含 `D2RMobile-Prepared.zip` 和 `Prepared-SHA256SUMS.txt`。将 ZIP 的地址和校验值填入 `prepared_input` JSON，日常运行 **Build IPA** 使用它。
