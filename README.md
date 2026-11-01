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
qualcommbe-ipq53xx-jdcloud_re-cs-08-squashfs-<类型>[-sfp-2.5]-Qin-owrt-<YYYYMMDD-HHMM>.bin
```

- `sfp-2.5`：光口 2.5G 版本（2500BASE-X 强制模式），10G 模块不可用
- `Qin`：构建标识
- `20261101-0450`：编译时刻（年月日-时分）

Release tag：`IPQ53XX-ER2-sfp-2.5-Qin-<YYYYMMDD-HHMM>`

刷机用 `factory.bin`，系统内升级用 `sysupgrade.bin`。

## 光口说明（重要）

**一、原生不工作**

上游代码直接编出来，光口插任何模块都是 NO-CARRIER。插 2.5G 猫棒时内核报：

```
sfp: autoneg setting not compatible with PCS
Module state: error
```

原因：`pcs-qcom-ipq9574.c` 的 `ipq_pcs_inband_caps()` 对 2500BASE-X 只声明了 `LINK_INBAND_DISABLE`，
而新版 phylink 在 SFP 插入时强制要求带内自协商，校验不通过就把模块判成 error、把 TX 禁掉。

**二、修复方式**

- 内核补丁 `0403-*`：PCS 不支持带内自协商时降级为强制模式，不再拒绝模块（**这条是通不通的关键**）
- 内核补丁 `0404-*`：模块读不到 A2h 软状态字节时停止轮询（已验证，日志从刷屏降到 1 条）
- DTS：`phy-mode` 由 `10gbase-r` 改为 `2500base-x`，对齐 PCS 实际能力（非关键，插模块后 phylink 会重新选）

补丁打在源码仓库 [LYOI/immortalwrt-sfp](https://github.com/LYOI/immortalwrt-sfp) 里，不在本仓库。

**三、已知遗留问题**

1. 光口与 LAN 之间的流量走 SoC 软件桥接，不享受 QCA8386 硬件转发，跑满 2.5G 时 CPU 占用会上去。
2. 插拔模块后偶发需要重启网口才会重新协商。
3. `phy-mode` 改动仅影响未插模块时的默认值，实际速率由模块 EEPROM 决定。

**四、10G 模块不可用**

- 本板 PCS 只实现 SGMII / SGMII+（2500BASE-X）/ USXGMII，没有 10GBASE-R，而 SFP 模块只会让它选 1000BASE-X / 2500BASE-X / 10GBASE-R
- OpenWrt 的 qca8k DSA 驱动 `mac_capabilities` 只到 `MAC_2500FD`
- 原厂 / jiang 的固件能跑 10G，是因为它们把 SFP 当作 QCA8386 的第 2 号端口（`media-type = "sfp"`）由 QSDK 驱动管理，硬件转发，两套架构不同
- 实测：插 10G AOC 时交换机侧 10G 口能 link up（说明光路与模块正常），但 ER2 侧 PCS 跑在 2.5G，无法同步，carrier 永远 0

结论：ImmortalWrt 下这个光口上限就是 2.5G。

**五、端口归属**

调试期：`sfp` 与 `lan1 / lan2 / lan3` 一起归 LAN，`wan` 电口仍作 WAN。
这样可以直接把终端插到光口上访问设备，方便验证。

需要把光口当 WAN 时，把 `02_network` 里那一行换成：

```sh
ucidef_set_interfaces_lan_wan "lan1 lan2 lan3 wan" "sfp"
```

完整调试过程见：[docs/ER2-SFP-notes.md](https://github.com/LYOI/immortalwrt-sfp/blob/owrt/docs/ER2-SFP-notes.md)

## 目录说明

- `workflows`：自定义 CI 配置
- `Scripts`：自定义脚本
- `Config`：自定义配置

## U-BOOT

高通版（沉心，推荐）：https://github.com/chenxin527/uboot-qsdk12.5-build.git

## 上游

配方与脚本源自 [VIKINGYFY/OpenWRT-CI](https://github.com/VIKINGYFY/OpenWRT-CI)，
源码源自 [VIKINGYFY/immortalwrt](https://github.com/VIKINGYFY/immortalwrt)。
