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

- `sfp-2.5`：历史遗留标识。这个后缀最早用来区分“光口只跑 2.5G”的版本，
  现在光口 10G / 2.5G 模块都能用，后缀不再代表速率上限
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

- 内核补丁 `0403-*`：PCS 不支持带内自协商时降级为强制模式，不再拒绝模块（**2.5G 模块通不通的关键**）
- 内核补丁 `0404-*`：模块读不到 A2h 软状态字节时停止轮询（已验证，日志从刷屏降到 1 条）
- 内核补丁 `0405-*`：10GBASE-R 补上 PCS `MISC2` 的 SerDes 模式位（**10G 模块通不通的关键**）
- DTS：`phy-mode` 由 `10gbase-r` 改为 `2500base-x`，插模块后由 phylink 重新选择

补丁打在源码仓库 [LYOI/immortalwrt-sfp](https://github.com/LYOI/immortalwrt-sfp) 里，不在本仓库。

**三、已知遗留问题**

1. 光口与 LAN 之间的流量走 SoC 软件桥接，不享受 QCA8386 硬件转发，跑满时 CPU 占用会上去。
2. 插拔模块后偶发需要重启网口才会重新协商。
3. 光口经 10G 交换机后，二层通（DHCP 能拿到租约、ARP 能解析）但三层单播不通；
   同一终端直连 `lan3` 电口则 0 丢包。已排除 NAS 侧配置、ER2 flow offloading、
   双网口同网段回包走错口、两台不同交换机与换端口换光纤位置，待用另一台 10G 设备做对照。

**四、10G 模块**

早期结论是“光口上限 2.5G”，那是误判。原因是驱动里 10GBASE-R 分支只写了 `PCS_MODE_XPCS`，
从没写过 `MISC2`：数字侧切到 XPCS 按 10.3125G 跑，模拟侧还停在上一次的 SGMII（0x3，1.25G），
两边速率对不上，link training 永远完不成，所以 carrier 恒为 0。

补丁 `0405-*` 把 10GBASE-R 折进 USXGMII 的 MISC2 赋值后：

```
PCS1 MISC2 0x30 → 0x70
XPCS KR_STS 0x0000000c → 0x0000100d (link=1)
sfp: Link is Up - 10Gbps/Full - flow control rx/tx
```

USXGMII 本身就是多速率 SerDes，同一个口 10G / 5G / 2.5G / 1G 都能自适应，
后续可以在 `pcs_config()` 里按 link speed 选 `SEL` 与 `MISC2`，做到插什么模块跑什么速率。

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
