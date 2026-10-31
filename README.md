# OpenWRT-CI（太乙 Plus ER2 / JDCloud RE-CS-08 专用）

本仓库只做一台机器：太乙 Plus ER2（JDCloud RE-CS-08）。

- 硬件：IPQ5332（四核 A53，WiFi7）+ QCA8386 交换芯片
- 平台：`qualcommbe` / `ipq53xx`，内核 6.18
- 源码：[LYOI/immortalwrt-sfp](https://github.com/LYOI/immortalwrt-sfp) 分支 `owrt`
- 配置：`Config/IPQ53XX-ER2.txt`（通用项在 `Config/GENERAL.txt`）
- 工作流：`.github/workflows/ER2-BUILD.yml`

## 编译

Actions → `ER2-BUILD` → `Run workflow`：

- `PACKAGE`：临时追加插件包，多个用换行分隔
- `TEST`：勾选后只输出 `.config`，不编译固件

## 产物命名

```
qualcommbe-ipq53xx-jdcloud_re-cs-08-squashfs-<类型>-sfp-2.5-Qin-owrt-<YYYYMMDD-HHMM>.bin
```

- `sfp-2.5`：光口 2.5G 版本（2500BASE-X 强制模式），10G 模块不可用
- `Qin`：构建标识
- `20261101-0450`：编译时刻（年月日-时分）

Release tag：`IPQ53XX-ER2-sfp-2.5-Qin-<YYYYMMDD-HHMM>`

刷机用 `factory.bin`，系统内升级用 `sysupgrade.bin`。

## SFP 光口说明

- 2.5G 模块（含 PON 猫棒）可用：2500BASE-X 强制模式，2.5Gbps / Full
- 10G 模块不可用：本板 PCS 只实现 SGMII / SGMII+(2500BASE-X) / USXGMII，不支持 10GBASE-R；
  且 OpenWrt 的 qca8k DSA 驱动 mac_capabilities 只到 MAC_2500FD
- 完整调试记录：[docs/ER2-SFP-notes.md](https://github.com/LYOI/immortalwrt-sfp/blob/owrt/docs/ER2-SFP-notes.md)

## 目录说明

- `workflows`：自定义 CI 配置
- `Scripts`：自定义脚本
- `Config`：自定义配置

## U-BOOT

高通版（沉心，推荐）：https://github.com/chenxin527/uboot-qsdk12.5-build.git

## 上游

配方与脚本源自 [VIKINGYFY/OpenWRT-CI](https://github.com/VIKINGYFY/OpenWRT-CI)，
源码源自 [VIKINGYFY/immortalwrt](https://github.com/VIKINGYFY/immortalwrt)。
