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

- 内核补丁 `0403-*`：PCS 不支持带内自协商时降级为强制模式，不再拒绝模块
- 内核补丁 `0404-*`：模块读不到 A2h 软状态字节时停止轮询
- DTS：`phy-mode` 由 `10gbase-r` 改为 `2500base-x`，对齐 PCS 实际能力

这两个补丁打在源码仓库 [LYOI/immortalwrt-sfp](https://github.com/LYOI/immortalwrt-sfp) 里，不在本仓库。

**三、已知遗留问题**

1. 部分模块（如诺基亚 3FE46541AA 猫棒）不实现 A2h 页软状态字节，未含 0404 补丁的固件会每秒刷约 20 条
   `failed to read SFP soft status: -ENXIO`，不影响链路，只是日志脏。
2. 插拔模块后偶发需要重启网口才会重新协商。
3. 光口与 LAN 之间的流量走 SoC 软件桥接，不享受 QCA8386 硬件转发，跑满 2.5G 时 CPU 占用会上去。

**四、10G 模块不可用**

- 本板 PCS 只实现 SGMII / SGMII+（2500BASE-X）/ USXGMII，没有 10GBASE-R，而 SFP 模块只会让它选 1000BASE-X / 2500BASE-X / 10GBASE-R
- OpenWrt 的 qca8k DSA 驱动 `mac_capabilities` 只到 `MAC_2500FD`，也无法把 SFP 交给交换芯片跑 10G
- 原厂能跑 10G 是因为 QSDK 把 SFP 当作 QCA8386 的第 2 号端口（`media-type = "sfp"`）硬件转发，两套架构不同

结论：ImmortalWrt 下这个光口上限就是 2.5G。

**五、端口归属**

`sfp` 归 WAN，`lan1 / lan2 / lan3` 和原来的 `wan` 电口全部归 LAN。

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
