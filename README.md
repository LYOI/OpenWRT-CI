# OpenWRT-CI（太乙 Plus ER2 专用）

太乙 Plus ER2 / JDCloud RE-CS-08 的云编译仓库。

- 硬件：IPQ5332（四核 A53，WiFi7），平台 `qualcommbe` / `ipq53xx`
- 源码：[LYOI/immortalwrt-sfp](https://github.com/LYOI/immortalwrt-sfp) 分支 `owrt`
- 配置：`Config/IPQ53XX-ER2.txt`
- 工作流：`.github/workflows/ER2-BUILD.yml`（手动触发，可只输出配置不编译固件）

# 编译

在 Actions 中选择 `ER2-BUILD` 执行 `Run workflow`：

- `PACKAGE`：临时追加插件包，多个用换行分隔
- `TEST`：勾选后只输出 `.config`，不编译固件

产物为 `factory.bin` 与 `sysupgrade.bin`，刷机用 `factory.bin`。

# 目录说明

- `workflows`——自定义 CI 配置
- `Scripts`——自定义脚本
- `Config`——自定义配置（`GENERAL.txt` 为通用项，`IPQ53XX-ER2.txt` 为设备项）

# U-BOOT

高通版（沉心，推荐）：

https://github.com/chenxin527/uboot-qsdk12.5-build.git

# 上游

配方与脚本源自 [VIKINGYFY/OpenWRT-CI](https://github.com/VIKINGYFY/OpenWRT-CI)，
源码源自 [VIKINGYFY/immortalwrt](https://github.com/VIKINGYFY/immortalwrt)。
