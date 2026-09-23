# OpenWrt 云编译（Lede / Lean OpenWrt 24.10）

基于 [coolsnowwolf/lede](https://github.com/coolsnowwolf/lede)（master 分支，当前默认版本 **24.10.5**，与你要求的 24.10.3 同属 24.10.x 系列且更新）的 GitHub Actions 云编译配置，一套仓库编译**两套互不混用的固件**：

| 固件 | 目标平台 | 适用设备 | 管理IP |
|---|---|---|---|
| `x86` | x86_64（内核 6.18） | Intel J1900 / J1800 软路由（通用 Generic 镜像） | `10.0.1.1` |
| `ax3000t` | mediatek/filogic（内核 6.12） | 小米 AX3000T（MT7981B，精简版） | `10.0.0.1` |

两者共同配置：主机名 `OpenWRT`、用户名 `root`、密码 `password`（Lede 默认即为此密码，此处显式固化）。

---

## 一、X86 全功能固件（`configs/x86.config`）

| 需求 | 实际包 | 来源 |
|---|---|---|
| passwall | `luci-app-passwall`（v26.9.1，含 xray + sing-box 核心） | **Lede luci feed 自带**（与 `Openwrt-Passwall/openwrt-passwall` 同源，避免重复冲突） |
| passwall2 | `luci-app-passwall2`（All 核心） | `Openwrt-Passwall/openwrt-passwall2` |
| passwall/passwall2 依赖 | v2ray-geoip、v2ray-geosite、ipt2socks、shadowsocks-rust 等 | `Openwrt-Passwall/openwrt-passwall-packages`（构建时自动去除与 feed 重复的 7 个子包：chinadns-ng/dns2socks/geoview/microsocks/sing-box/tcping/xray-core） |
| 易有云文件管理器 | `luci-app-linkease` + **守护进程 `linkease`**（二进制来自 istoreos 官方） | `linkease/luci-app-linkease` + `linkease/nas-packages`（自行找到） |
| 1Panel | 无 luci 插件 → 见下方【1Panel 说明】 | — |
| 全能推送 | `luci-app-pushbot` | `zzsj0928/luci-app-pushbot`（你清单外，自行找到） |
| 微信推送 | `luci-app-wechatpush`（v3.6.12，即 tty228 的 ServerChan 项目） | **Lede luci feed 自带**（避免与 `tty228/luci-app-serverchan` 重复冲突） |
| DDNS-GO | `ddns-go` + `luci-app-ddns-go` | **Lede packages/luci feed 自带**（sirpdboy 项目已并入 feed，避免重复克隆冲突） |
| Lucky | `lucky` + `luci-app-lucky` | **Lede packages/luci feed 自带**（同上） |
| OAF 行为管理 | `luci-app-appfilter` + `appfilter` + `kmod-oaf` | **Lede feeds 自带**（外部 `destan19/OpenAppFilter` 与 feed 同名冲突，故不克隆） |
| uhttpd | `luci-app-uhttpd` | Lede luci feed 自带 |
| 负载均衡 | `luci-app-mwan3` | Lede luci feed 自带 |
| TTYD 终端 | `luci-app-ttyd` | Lede luci feed 自带 |
| 磁盘管理 | `luci-app-diskman` | Lede luci feed 自带 |
| 释放内存 | `luci-app-ramfree` | Lede luci feed 自带 |
| Argon 主题 | `luci-theme-argon` | Lede luci feed 自带（默认启用，首启自动切换） |
| Argon 设置 | `luci-app-argon-config` | Lede luci feed 自带 |
| iStore 商店（含中文包） | `luci-app-store` + `luci-lib-taskd` + `taskd` + `luci-lib-xterm` | `linkease/istore`（界面内置简体中文） |
| 自定义命令 | `luci-app-commands` | Lede luci feed 自带 |
| Docker（概览/容器/镜像/网络/存储卷/事件/配置） | `luci-app-docker` + `docker` + `dockerd` | Lede luci feed + packages feed 自带 |
| 网络共享 | `luci-app-samba4` | Lede luci feed 自带 |
| alist 文件列表 / openlist | `luci-app-openlist`（OpenList 是 Alist 的继任者，Makefile 声明 `PKG_PROVIDES:=luci-app-alist`） | Lede luci feed 自带 |
| 流量统计 | `luci-app-nlbwmon` | Lede router 默认内置 |
| TurboACC（BBR 最新） | `luci-app-turboacc`（Flow offloading + BBR CCA） | Lede router 默认内置 |
| IP 限速 | `luci-app-eqosplus` | `sirpdboy/luci-app-eqosplus` |

**中文语言包：** OpenWrt 24.10 / Lede 的 LuCI 语言包命名规范为 `luci-i18n-<插件>-zh_Hans`（po 目录 `zh-cn` 会自动映射为 `zh_Hans`），上述所有带中文包的插件均已配置安装，未配置的插件则无中文包。

**关于"与自带基础插件重复"的处理（你要的说明）：**
- `流量统计`、`TurboACC`、`luci`、`ssr-plus` 等本就是 Lede 的 **router 默认内置插件**（`include/target.mk` 的 `DEFAULT_PACKAGES.router`），不是重复添加——它们会随固件默认编译进去，无需、也无法通过 `.config` 删除（符合"其他插件不要删除"）。
- `软件包管理`（原 luci-app-opkg）在 OpenWrt 24.10 已**并入 luci 本体**（luci-mod-system，菜单「系统 → 软件包」），无需单独插件。
- `iStore 商店`界面自带简体中文（istore-ui 内置 zh-cn 翻译），无需单独语言包。
- `luci-app-openlist` 与 `luci-app-alist` 是同一项目前后身（openlist 的 Makefile 声明 `PROVIDES: luci-app-alist`），只装 openlist 一个即覆盖"alist 文件列表 + openlist"两项要求。
- OAF 内核模块已做 Linux 6.18 兼容补丁（`del_timer_sync`→`timer_delete_sync`，见 `.github/patches/oaf-kernel-6.18-timer.patch`，构建时自动应用），保证 OAF 行为管理在 6.18 内核可用。
- `ddns-go`、`lucky`、`微信推送(wechatpush)` 均已被 Lede feeds 收录，外部同名仓库不再克隆（克隆会因包名重复导致构建失败）。
- 你清单里其余仓库（themes、clash、dae、adguardhome、easytier 等）不属于上述必装清单，未启用（不删 lede 自带项，也不会误装）。

**1Panel 说明：**
- 经核查 GitHub/Gitee **不存在 luci-app-1panel 插件**（开源社区没有这个包）。
- 本固件已内置 **Docker**，推荐在 Docker 里运行 1Panel（官方一键安装脚本，OpenWrt x86 上可直接执行）：
  ```bash
  curl -sSL https://resource.fit2cloud.com/1panel/package/quick_start.sh -o quick_start.sh && sh quick_start.sh
  ```
- 若想要适配 OpenWrt 的 1Panel 二进制，可参考社区项目 `gcsong023/wrt1panel`。

## 二、小米 AX3000T 精简固件（`configs/ax3000t.config`）

仅包含（与 X86 完全独立，互不混用）：
- **基础功能**（Luci、防火墙、opkg，含中文语言包）
- **WiFi 2.4G / 5G**：设备默认包 `kmod-mt7981-firmware` / `mt7981-wo-firmware`，信道、频段带宽在 LuCI「无线」页面直接修改
- **IP 限速**：`luci-app-eqosplus`
- **passwall**（精简：仅内置 sing-box 核心，xray 等核心运行时可在线下载，适配 128MB 闪存；同时添加 passwall-packages 补齐硬依赖 ipt2socks 等）
- **TTYD 终端**：`luci-app-ttyd`
- **TurboACC（最新 BBR）**：`luci-app-turboacc`

**Breed / 不死引导（小米用）说明：**
- AX3000T 没有 hackpascal 的 Breed（Breed 不支持 MT7981），社区通用替代是 **hanwckf/bl-mt798x 不死 U-Boot**（相当于 Breed 的 Web 恢复控制台）。
- 每次 AX3000T 构建的 Release 会自动附带 bl-mt798x 最新发布包；推荐文件 `mt7981_ax3000t-fip-fixed-parts-multi-layout.bin`。
- 刷入（需先开启官方固件 SSH）：`mtd write /tmp/<uboot文件> FIP`，之后按住 RESET 上电进入 U-Boot Web 恢复界面。
- 参考：https://github.com/hanwckf/bl-mt798x ；官方刷机 Wiki：https://openwrt.org/toh/xiaomi/xiaomi_mi_router_ax3000t

## 三、自动清理与自动构建（`auto-clean-build.yml`）

| 时间（北京时间） | 动作 |
|---|---|
| 03:45（=UTC 19:45） | 清理 Release：**只保留最近 15 个** → 清理成功 → **连锁触发** X86 与 AX3000T 构建 |
| 03:50（=UTC 19:50） | 兜底：检查 03:45 清理流程的运行结果 → **若清理失败或未运行 → 自动触发构建**（保证每天都有新固件） |

> GitHub Actions 的 `schedule` 按 **UTC** 执行，故 03:45/03:50 北京时间对应 `45 19 * * *` / `50 19 * * *`。

## 四、手动使用

1. 仓库根目录 → **Actions** 页 → 选择 `Build OpenWrt X86` 或 `Build OpenWrt AX3000T` → **Run workflow** 即可手动云编译。
2. 编译完成后固件自动上传到 **Releases**（x86 附 img.gz 镜像；ax3000t 附 sysupgrade.bin 与不死U-Boot）。
3. 修改 `configs/*.config` 或 `files/*` 并 push 到 main 分支也会自动触发对应构建。

## 五、文件结构

```
.github/workflows/
  build-x86.yml          # X86_64 全功能构建
  build-ax3000t.yml      # 小米 AX3000T 精简构建
  auto-clean-build.yml   # 每日 03:45 清理+连锁构建 / 03:50 兜底构建
configs/
  x86.config             # X86 全功能配置
  ax3000t.config         # AX3000T 精简配置
files/
  x86/                   # X86 首启设置（主机名/IP/密码）
  ax3000t/               # AX3000T 首启设置（主机名/IP/密码）
```
