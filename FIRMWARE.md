# 腾达 BE12 Pro · ImmortalWrt 固件使用说明

> 编译日期：2026-10-28 ｜ 内核：6.12.112 ｜ 版本号：r0-4ab9dab ｜ 软件包：367 个
> 源码：`chasey-dev/immortalwrt-mt798x-rebase`（25.12-dev-wifi7 分支）
> 构建：GitHub Actions `zxc4415/immortalwrt-be12-pro` #40715509076

---

## 一、这是什么

腾达 BE12 Pro 的第三方 OpenWrt 固件（ImmortalWrt 分支），基于 GitHub 开源项目
`l3692847/immortalwrt-be12-pro` 修复后重新编译。

原项目有两处长期存在的问题，本版均已修复：

| 问题 | 原项目 | 本版 |
|---|---|---|
| **iStore 软件中心** | 配置里写了但从未真正编入（连续多版产物均缺失） | ✅ 完整可用的软件中心 |
| **Clash 内核** | 只有 OpenClash 插件外壳，内核需联网另下 | ✅ 已内置 `mihomo` + `v2ray-geoip`，装好即用 |

### 修复原因（给想自己编译的人）

原项目 iStore 编不进去，是一个很隐蔽的 kconfig 依赖问题：

```
luci-app-store 的 Makefile 里有
  LUCI_DEPENDS := ... +!(PACKAGE_TAR_XZ):xz-utils
```

固件里 `tar` 默认带 XZ 支持（`PACKAGE_TAR_XZ=y`），但 `xz-utils` 没被选中，
依赖条件不成立 → kconfig 把整个 `luci-app-store` 符号**隐藏**掉。
表现就是 `.config` 里连 `# CONFIG_PACKAGE_luci-app-store is not set` 都不出现，
`make defconfig` 静默丢弃，编译日志里也几乎没有痕迹。

**修复**：`config/defconfig-delta` 增加 `CONFIG_PACKAGE_xz-utils=y`；
同时补齐 `mihomo`、`v2ray-geoip`、`taskd` 依赖链，
并在工作流里加入"编译前校验必需包是否真被选中"的强制门禁，避免再次静默失败。

---

## 二、硬件与版本信息

| 项目 | 参数 |
|---|---|
| 型号 | Tenda BE12 Pro |
| SoC | MediaTek MT7987A（四核 ARM64） |
| 无线 | MT7992，Wi-Fi 7（802.11be），2.4G / 5G / 6G |
| 内存 | 512 MB |
| 内核 | 6.12.112 |
| 固件版本 | r0-4ab9dab |
| 默认管理地址 | **192.168.3.1** |
| 默认账号 | `root`（无初始密码，首次登录时设置） |
| 包管理器 | opkg（367 个包） |

### 已内置组件

| 组件 | 版本 | 说明 |
|---|---|---|
| OpenClash | 0.48.002 | 代理客户端（LuCI 服务菜单） |
| mihomo | 1.19.32-r1 | Clash 内核，**已预装，无需联网下载** |
| v2ray-geoip / geosite | 202610050438 | 分流用的地理位置数据库 |
| iStore | luci-app-store 0.2.1-r1 | 软件中心（LuCI 系统菜单） |
| taskd / luci-lib-taskd / luci-lib-xterm | 1.0.3 / 1.0.26 / 4.18.0 | iStore 依赖链 |
| LuCI 中文语言包 | 26.298.29275 | 完整简体中文 |
| Argon 主题 | 2.4.7 | 界面主题 |
| Wi-Fi 7 驱动 | kmod-mt7992 / mt_wifi7 / mt_hwifi | 6.12.112.2025.06.13 |

---

## 三、文件说明

| 文件 | 大小 | 用途 |
|---|---|---|
| `immortalwrt-mediatek-filogic-tenda_be12-pro-squashfs-sysupgrade.bin` | 50 MB | **刷机主文件**，网页升级 / sysupgrade 使用 |
| `immortalwrt-mediatek-filogic-tenda_be12-pro-initramfs-kernel.bin` | 44 MB | 内存运行系统，**TFTP 救砖 / 首次试水**用（不写 flash） |
| `SHA256SUMS.txt` | 265 B | 校验值 |
| `manifest.txt` | 11 KB | 完整包清单（367 个包） |
| `config.buildinfo` | 14 KB | 本次编译的完整 kconfig |
| `feeds.buildinfo` | 671 B | 本次编译使用的 feed 源 |

### 下载后务必先校验

**Windows PowerShell**

```powershell
Get-FileHash .\immortalwrt-mediatek-filogic-tenda_be12-pro-squashfs-sysupgrade.bin -Algorithm SHA256
```

**macOS / Linux**

```sh
shasum -a 256 immortalwrt-mediatek-filogic-tenda_be12-pro-squashfs-sysupgrade.bin
```

应当与下面完全一致：

```
dad8d136a3c9b9ef2ce83d4ff85d18a898d66265ff9af13f6074b0b0ae1e3b08  sysupgrade
351d6343401e2f7eaac5d88d85f6688110a83e18c0b0753a9ee56cc3b72dc9cf  initramfs
```

不一致就不要刷——说明文件下载损坏。

---

## 四、刷机

### 情况 A：路由器当前已是 OpenWrt / ImmortalWrt

最简单，网页升级即可。

1. 浏览器打开 `http://192.168.3.1`（若你改过 IP，用当前 IP）
2. 进入 **系统 → 备份 / 刷写固件 → 刷写新固件**
3. 选择 `...squashfs-sysupgrade.bin`
4. **取消勾选「保留当前配置」**（跨版本升级建议全新配置，避免旧配置冲突）
5. 确认刷写，等待自动重启（约 2 ~ 4 分钟）

刷完 IP 会回到 `192.168.3.1`，用户名 `root`，密码留空进入后立即设置。

### 情况 B：路由器还是腾达原厂固件（首次刷入）

> ⚠️ **绝对不能通过原厂网页上传 OpenWrt 固件，会直接变砖。**

原厂固件只能通过 **TFTP 引导** 或 **编程器/串口** 进入，不同生产批次的
按键时序、默认 IP 可能不一样。务必先看原帖确认你这一批的进入方式：

- 原帖：<https://www.right.com.cn/forum/thread-8477866-1-1.html>

**推荐的安全流程（先内存、后 flash）：**

1. **先 TFTP 推 `initramfs-kernel.bin`**
   这是纯内存系统，断电即消失，**不会写入 flash**，刷坏了拔电重启就回到原厂。
   - 电脑网卡设静态 IP（原帖会写明，常见是 `192.168.1.2/24` 或 `192.168.0.2/24`）
   - 用 TFTP 服务端（Windows 可用 Tftpd64）把文件放到根目录
   - 路由器断电 → 按住复位键上电 → 进入恢复模式 → 固件自动被拉取
2. 内存系统起来后（浏览器进管理页），**先验证**：
   - Wi-Fi 7 是否正常（2.4G / 5G / 6G 都能搜到并连上）
   - 网口是否全部可用
   - LED、按键是否正常
3. 确认没问题，再在内存系统里 **系统 → 备份/刷写固件** 上传
   `...squashfs-sysupgrade.bin`，写入 flash，正式落地。

> 🔴 **刷机前必须先备份原厂 ART / EEPROM 分区。**
> 这是无线校准数据（每台机器独有），丢了之后 Wi-Fi 功率、频段、速率会永久异常，
> 且无法通过重刷固件恢复。原厂系统下若有备份入口，先备份；进内存系统后
> 也可以用 `dd` 把 mtd 分区整体读出保存。

---

## 五、首次开机配置

1. 电脑/手机连上路由器（有线优先），浏览器打开 `http://192.168.3.1`
2. 首次登录 `root`，**立即设置密码**
3. 进入 **网络 → 接口 → WAN**，按你的上网方式配置（PPPoE 拨号 / DHCP / 静态 IP）
4. **网络 → 无线**，设置 SSID 与密码；6G 频段建议单独命名，方便区分
5. （可选）**系统 → 系统 → 时间同步**，确认时区为 `Asia/Shanghai`

### OpenClash 使用要点

首次进入 **服务 → OpenClash** 时，注意几项：

- **代理模式保持「规则」** —— 这是"国内直连、国外走代理"的关键；切成"全局"会让国内网站也绕出国
- **区域绕过选「大陆」** —— 国内流量不经过代理内核，省 CPU、降延迟
- **不要填「自定义规则」时带 `##`** —— `##` 在 YAML 里是注释，会让规则整条失效
- 内核已内置 `mihomo`，**不需要**再去"版本更新"里下载内核

给常用站点强制走代理的规则示例（覆写设置 → 规则设置 → 自定义规则）：

```yaml
- DOMAIN-SUFFIX,bing.com,自动选择
- DOMAIN-SUFFIX,copilot.microsoft.com,自动选择
```

> 组名（`自动选择`、`故障转移` 等）必须与订阅配置里的策略组名**完全一致**，
> 可在「控制面板 → 代理」页面查看。名字写错会导致 `proxy [...] not found`，
> 整份配置加载失败。

### iStore 使用

进入 **系统 → iStore**（或 LuCI 首页的软件商店图标），可直接搜索安装插件。
`taskd` 服务已预装并随系统启动，无需额外配置。

---

## 六、刷完验证

SSH 登录（`ssh root@192.168.3.1`）后逐条执行：

```sh
# 1. 机型识别是否正确（应输出 tenda,be12-pro）
ubus call system board

# 2. 看网络接口与无线接口
ip link

# 3. Wi-Fi 7 驱动与硬件加速是否加载
logread | grep -iE 'mt7992|hnat|wifi7'

# 4. 无线状态
iwinfo

# 5. 关键组件是否齐全
opkg list-installed | grep -E 'openclash|store|mihomo|taskd'

# 6. 内核版本
uname -r
```

LuCI 界面侧确认：

- **服务 → OpenClash** 状态显示"Meta 运行中"
- **系统 → iStore** 能正常打开、能搜索插件
- **状态 → 概览** 显示 367 个包、内核 6.12.112

---

## 七、常见问题

**Q：Bing 被误判成国内流量直连，打不开国际版怎么办？**
区域绕过选「大陆」时，Bing / 微软在国内有 CDN，会被当成大陆流量。
按上文在「自定义规则」里加一条 `DOMAIN-SUFFIX,bing.com,策略组名` 即可。

**Q：加了规则后 Bing 反而打不开了？**
多半是策略组里挑到了带「流媒体 / 媒体」字样的节点（这类节点只放行特定域名）。
换成 `故障转移` 组，或手动在组里锁定一个普通高速节点。

**Q：OpenClash 修改配置后保存报错 / 内核起不来？**
检查「自定义规则」框里是否混入了 `rule-providers:`、`rules:` 这类顶层键——
那个框只接受 `- 类型,匹配内容,目标` 格式的规则列表。

**Q：Google 能开但首屏很慢？**
「运行模式」增强模式只接管 TCP，QUIC（UDP 443）不进代理，浏览器要先超时再回退。
可切到 **TUN 模式**（本固件已含 `kmod-tun`），代价是 CPU 占用略升。

**Q：iStore 打不开 / 一直转圈？**
SSH 执行 `/etc/init.d/taskd status`，若未运行则 `/etc/init.d/taskd enable && /etc/init.d/taskd start`。

**Q：刷完进不去后台？**
先在电脑上 `ipconfig` 确认拿到 `192.168.3.x` 段地址；若仍不通，用 initramfs 版本
TFTP 重新引导，检查是否刷了错误的 sysupgrade 文件。

---

## 八、救砖

1. 断电，网卡设静态 IP（同首次 TFTP 时的设置）
2. 启动 TFTP 服务端，把 `initramfs-kernel.bin` 放根目录
3. 按住复位键上电，等固件被拉取、内存系统启动
4. 从内存系统里重新 sysupgrade 正确固件

只要 bootloader 没被破坏，这个流程都能救回来。**刷机时不要断电**，
尤其不要在原厂固件刷写过程中断电。

---

## 九、风险与免责

- 本固件为**第三方个人编译**，基于开源项目修复重编，**非厂商官方发布**。
- 刷机存在变砖风险，**请自行评估并承担后果**。
- 刷机前务必备份 ART / EEPROM 分区与原有配置。
- 512 MB 内存机型不建议安装重型插件（Docker、大型数据库等）。
- 上游项目规模很小（0 star），建议先在原帖确认同机型实际刷机反馈。
- 本固件不含任何商业授权组件；OpenClash / mihomo / iStore 均遵循其各自开源协议。

---

## 十、自己重新编译

编译配置与修复已固化在 fork 仓库中：
<https://github.com/zxc4415/immortalwrt-be12-pro>

在该仓库 **Actions → Build ImmortalWrt → Run workflow** 即可重编，
工作流已包含：

- iStore 依赖链自动安装与去重（避免与 feed 内同名包冲突）
- `xz-utils` 依赖修复
- mihomo / v2ray-geoip 显式打包
- **编译前强制校验**：必需包未被选中时直接失败，不会白等两小时

如需改配置，编辑 `config/defconfig-delta` 后重新触发即可，全程约 2 ~ 3 小时。
