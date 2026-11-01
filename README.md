# OpenWRT-CI（太乙 Plus ER2 / JDCloud RE-CS-08 专用）

> **归档状态：项目已归档（2026-11-02）。本仓库保留 ER2 编译流程作为存档，SFP 光口数据面转发无解，详见下方「光口说明」。**

本仓库只做一台机器：太乙 Plus ER2（JDCloud RE-CS-08）。

- 硬件：IPQ5332（四核 A53，WiFi7）+ QCA8386 交换芯片
- 平台：`qualcommbe` / `ipq53xx`，内核 6.18
- 源码：[LYOI/immortalwrt-sfp](https://github.com/LYOI/immortalwrt-sfp) 分支 `owrt`（同样已归档）
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

- `sfp-2.5`：历史遗留标识
- `Qin`：构建标识
- `20261101-0450`：编译时刻（年月日-时分）

Release tag：`IPQ53XX-ER2-sfp-2.5-Qin-<YYYYMMDD-HHMM>`

刷机用 `factory.bin`，系统内升级用 `sysupgrade.bin`。

## 光口说明（最终结论）

### 成果：速率自适应（link 层）

SFP 光口能自动识别模块速率并 link：
- 2.5G 模块 → 2.5Gbps/Full
- 10G 模块 → 10Gbps/Full

依赖三个内核补丁（在源码仓库 immortalwrt-sfp 里）：
- `0403-*`：PCS 带内自协商不支持时降级为强制模式
- `0404-*`：模块无 A2h 软状态时停止轮询
- `0405-*`：10GBASE-R 补上 PCS MISC2 的 SerDes 模式位

### 无解：光口↔电口数据面转发（跨界）

sfp 挂在 SoC PPE 直连口，lan/wan 是 QCA8386 DSA 口，两者不同硬件域。
上游 `qcom_ppe` 驱动对「PPE 直连口的 host-bound RX 交付」存在未完成 gap：

- 广播/ARP/DHCP 能过
- 单播 TCP 不通（SYN 无回程）
- ping 通但 SSH/HTTP 断

这是上游驱动的架构级 gap，不是单个 bug，本项目不做深挖。

**正确方向**：QSDK 架构（sfp 挂 QCA8386，SSDK 硬件转发），
现成 ER2 构建方案见 https://github.com/zt406600/wlan-ap（er2 分支）。

完整调试记录见：[docs/ER2-SFP-notes.md](https://github.com/LYOI/immortalwrt-sfp/blob/owrt/docs/ER2-SFP-notes.md)

## 目录说明

- `workflows`：自定义 CI 配置
- `Scripts`：自定义脚本
- `Config`：自定义配置

## U-BOOT

高通版（沉心，推荐）：https://github.com/chenxin527/uboot-qsdk12.5-build.git

## 上游

配方与脚本源自 [VIKINGYFY/OpenWRT-CI](https://github.com/VIKINGYFY/OpenWRT-CI)，
源码源自 [VIKINGYFY/immortalwrt](https://github.com/VIKINGYFY/immortalwrt)。
