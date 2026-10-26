# Cudy TR3000 FanchmWrt 二次开发与 AI 编译指南

本文档用于在 Linux 设备上，让 AI 基于当前仓库修改、配置并编译适用于 **Cudy TR3000 v1** 的 FanchmWrt 固件。

> 本项目使用第三方“不死 U-Boot”的 **112M 布局**。固件只应更新系统 UBI 分区，不得覆盖 BL2、U-Boot 环境、Factory、bdinfo 或 FIP。

## 1. 已确认的路由器信息

| 项目 | 实测值 |
| --- | --- |
| 设备 | Cudy TR3000 v1 |
| SoC | MediaTek MT7981，双核 ARMv8 |
| 内存 | 512 MB，系统可用约 479 MB |
| 闪存 | 128 MB SPI NAND |
| 当前设备标识 | `cudy,tr3000-mod` |
| 当前固件 | Kwrt 24.10-SNAPSHOT，Linux 6.6.118 |
| 管理地址 | `10.0.0.1` |
| U-Boot 恢复地址 | `192.168.1.1` |
| U-Boot | U-Boot 2022.07-rc3，构建标识 `Cudy_Tr3000_v1_uboot_25.11.21` |
| 恢复页面布局 | `112m` |

实测 MTD 布局：

| 分区 | 起始地址 | 大小 | 用途 |
| --- | ---: | ---: | --- |
| `mtd0` BL2 | `0x000000` | 1 MiB | 启动前级 |
| `mtd1` u-boot-env | `0x100000` | 512 KiB | U-Boot 环境变量 |
| `mtd2` Factory | `0x180000` | 2 MiB | Wi-Fi 校准等设备数据 |
| `mtd3` bdinfo | `0x380000` | 256 KiB | MAC 地址等板级信息 |
| `mtd4` FIP | `0x3c0000` | 2 MiB | ATF/U-Boot 固件包 |
| `mtd5` ubi | `0x5c0000` | 112 MiB | Linux 内核和根文件系统 |

当前 U-Boot 网页恢复流程已经实际验证：电脑设置 `192.168.1.2/24`，进入 `http://192.168.1.1` 后可看到“固件更新”页面和 `112m` 布局选项，正常重启后路由器仍可返回 `10.0.0.1`。

## 2. 源码位置和当前改动

仓库目录：

```text
Wan_op_system/fanchmwrt
```

当前分支：

```text
fanchmwrt-25.12.6
```

当前源码已新增独立设备目标：

```text
cudy_tr3000-mod
```

对应文件：

```text
target/linux/mediatek/dts/mt7981b-cudy-tr3000-mod.dts
target/linux/mediatek/image/filogic.mk
target/linux/mediatek/filogic/base-files/etc/board.d/02_network
target/linux/mediatek/filogic/base-files/etc/hotplug.d/ieee80211/11_fix_wifi_mac
target/linux/mediatek/base-files/lib/preinit/05_set_preinit_iface
```

该目标的关键参数：

```make
DEVICE_DTS := mt7981b-cudy-tr3000-mod
SUPPORTED_DEVICES += R47 cudy,tr3000-mod
BLOCKSIZE := 128k
PAGESIZE := 2048
IMAGE_SIZE := 114688k
KERNEL_IN_UBI := 1
IMAGE/sysupgrade.bin := sysupgrade-tar | append-metadata
```

设备树中的 UBI 分区必须保持：

```dts
&ubi {
    reg = <0x5c0000 0x7000000>;
};
```

不要将该目标替换成仓库已有的 `cudy_tr3000-v1-ubootmod`。那个目标使用 OpenWrt FIT/ITB U-Boot 布局，生成 `sysupgrade.itb`，与当前第三方多布局 U-Boot 的 112M 系统布局不同。

## 3. 给 Linux AI 的强制约束

开始工作前，把以下内容作为 AI 的任务约束：

```text
你正在为 Cudy TR3000 v1 编译 FanchmWrt。

硬件和布局：
- SoC：MediaTek MT7981
- RAM：512 MB
- SPI NAND：128 MB
- board_name：cudy,tr3000-mod
- UBI：起始 0x5c0000，大小 0x7000000（112 MiB）
- 使用第三方网页恢复 U-Boot

必须遵守：
1. 目标必须是 cudy_tr3000-mod。
2. 产物必须是 squashfs-sysupgrade.bin。
3. 不得生成或刷写 BL2、FIP、preloader、U-Boot 或 Factory 镜像。
4. 不得改变 UBI 起始地址和 112M 大小。
5. 不得选用 cudy_tr3000-v1-ubootmod 的 sysupgrade.itb。
6. 修改前检查 git diff，修改后运行 git diff --check。
7. 编译失败时先保留完整日志，再进行最小修复。
8. 不得自动刷写路由器。只编译、校验并报告产物路径和哈希。
9. 不得把密钥、拨号密码、无线密码或设备备份提交到 Git。
```

## 4. Linux 编译环境

仓库 README 推荐 Ubuntu 22.04。建议使用 Ubuntu 22.04 LTS x86_64，至少准备：

- 4 核 CPU，建议 8 核以上
- 8 GB RAM，建议 16 GB
- 60 GB 可用磁盘，功能较多时建议 100 GB
- 稳定互联网连接
- 普通非 root 用户执行编译

Ubuntu 22.04 可安装以下依赖：

```bash
sudo apt update
sudo apt install -y \
  build-essential clang flex bison g++ gawk gcc-multilib gettext git \
  libncurses-dev libssl-dev python3 python3-dev python3-setuptools \
  rsync swig unzip zlib1g-dev file wget curl subversion libelf-dev \
  liblzma-dev libzstd-dev python3-pyelftools python3-packaging ccache
```

检查环境：

```bash
git --version
make --version
gcc --version
python3 --version
df -h .
```

OpenWrt 不应以 root 身份编译：

```bash
test "$(id -u)" -ne 0 || { echo "不要使用 root 编译"; exit 1; }
```

## 5. 获取并检查源码

如果通过 Git 传输当前工作树：

```bash
git clone <仓库地址> fanchmwrt
cd fanchmwrt
git checkout fanchmwrt-25.12.6
git status --short --branch
```

确认 112M 目标已经存在：

```bash
test -f target/linux/mediatek/dts/mt7981b-cudy-tr3000-mod.dts
grep -n "Device/cudy_tr3000-mod" target/linux/mediatek/image/filogic.mk
grep -n "0x5c0000 0x7000000" target/linux/mediatek/dts/mt7981b-cudy-tr3000-mod.dts
```

如果通过压缩包或文件同步传输，先确认执行位和符号链接没有丢失：

```bash
git diff --summary
git diff --check
```

## 6. 安装 feeds

首次编译时执行：

```bash
./scripts/feeds update -a
./scripts/feeds install -a
```

`feeds.conf.default` 已固定 OpenWrt 官方 feeds 的提交版本，同时引用：

```text
https://github.com/fanchmwrt/fanchmwrt-packages.git
```

FanchmWrt 的具体 LuCI 页面很可能来自该 feed。主题外壳和 `fwxd` 后端位于主仓库中，但 `fwx_dashboard`、`fwx_network` 等完整页面在未安装 feeds 时并不完整。

安装 feeds 后检查：

```bash
find package/feeds -maxdepth 3 -type d -iname '*fwx*' -o -iname '*fanchm*'
grep -R "fwx_dashboard" -n package/feeds feeds 2>/dev/null | head -50
```

## 7. 配置设备目标

先写入最小目标配置：

```bash
cat > .config <<'EOF'
CONFIG_TARGET_mediatek=y
CONFIG_TARGET_mediatek_filogic=y
CONFIG_TARGET_mediatek_filogic_DEVICE_cudy_tr3000-mod=y
CONFIG_TARGET_ROOTFS_SQUASHFS=y
EOF

make defconfig
```

验证唯一设备目标：

```bash
grep '^CONFIG_TARGET_mediatek_filogic_DEVICE_.*=y' .config
```

预期至少包含：

```text
CONFIG_TARGET_mediatek_filogic_DEVICE_cudy_tr3000-mod=y
```

如果使用交互配置：

```bash
make menuconfig
```

选择：

```text
Target System  ---> MediaTek Ralink ARM
Subtarget      ---> Filogic 8x0
Target Profile ---> Cudy TR3000 v1 (112M U-Boot layout)
```

## 8. 推荐功能组合

TR3000 的 512 MB RAM 足以运行常用路由服务，但内部闪存仍应保留升级和配置空间。

第一版建议包含：

- LuCI 中文界面
- FanchmWrt 主题
- `fwx`、`fwxd` 家长控制和设备管理
- nftables Full Cone NAT
- MediaTek 硬件流量卸载
- WireGuard
- DDNS
- UPnP
- SQM
- SmartDNS 或 MosDNS（二选一作为主要上游解析器）
- AdGuard Home（需要合理安排 DNS 监听端口）
- USB 存储和 Samba
- 实时流量、温度和设备状态

代理插件建议只选择一套：

- OpenClash
- PassWall
- HomeProxy
- SSR Plus

DNS 服务不要全部直接监听 53 端口。可采用如下链路：

```text
客户端:53 -> AdGuard Home:53 -> SmartDNS:6053 -> 上游 DNS
```

150 Mbps 线路建议启用 SQM 时使用 CAKE，并将上下行设置为实测速率的约 90%～95%。硬件流量卸载与 SQM 可能互相绕过，最终配置应根据“低延迟”或“最高吞吐”二选一。

## 9. 原固件 UI 架构

主仓库中的 UI 使用 LuCI JavaScript、ucode 模板、rpcd 和 ubus：

```text
浏览器 LuCI 页面
    -> rpc.declare()
    -> rpcd ACL
    -> ubus fwx.common
    -> fwxd
    -> UCI / 内核 / 网络接口
```

主题相关源码：

```text
package/fcm/luci-theme-fanchmwrt/ucode/template/themes/fanchmwrt/header.ut
package/fcm/luci-theme-fanchmwrt/htdocs/luci-static/resources/menu-fanchmwrt.js
package/fcm/luci-theme-fanchmwrt/htdocs/luci-static/fanchmwrt/cascade.css
package/fcm/luci-theme-fanchmwrt/htdocs/luci-static/fanchmwrt/mobile.css
```

原菜单支持：

- 控制台
- 网络设置
- 网络服务
- 无线设置
- 家长控制
- 高级设置
- 上网记录
- 用户管理
- 状态监控
- 应用中心
- 网络工具
- 游戏加速
- 云服务
- AC 管理

页面开发应继续使用 LuCI 的 `view.extend()`、`form.Map()`、`rpc.declare()` 和 `uci` API，不应把独立静态原型直接覆盖到 `/www`。

## 10. 下载源码包与编译

先下载所有构建依赖并显示失败项：

```bash
make download -j8
find dl -type f -size -1024c -print
```

如果发现异常的小文件，检查后删除对应文件并重新执行下载。

首次编译建议保留详细日志：

```bash
mkdir -p logs
make -j"$(nproc)" V=s 2>&1 | tee logs/build-tr3000.log
```

若并行构建失败，使用单线程定位真实错误：

```bash
make -j1 V=s 2>&1 | tee logs/build-tr3000-retry.log
```

不要看到第一个 `Error 2` 就直接修改代码，应向上查找最早出现的编译器、补丁或依赖错误。

## 11. 查找与校验固件

构建完成后：

```bash
find bin/targets/mediatek/filogic -maxdepth 1 -type f \
  -name '*cudy_tr3000-mod*' -printf '%f %s bytes\n'
```

目标文件名应类似：

```text
fanchmwrt-mediatek-filogic-cudy_tr3000-mod-squashfs-sysupgrade.bin
```

不得使用这些文件：

```text
*preloader.bin
*bl31-uboot.fip
*sysupgrade.itb
*factory.bin（除非以后针对当前 U-Boot 明确实现并验证）
```

生成校验值：

```bash
cd bin/targets/mediatek/filogic
sha256sum *cudy_tr3000-mod*squashfs-sysupgrade.bin \
  | tee cudy_tr3000-mod-sha256.txt
```

检查镜像元数据：

```bash
./scripts/json_overview_image_info.py \
  bin/targets/mediatek/filogic/*.manifest 2>/dev/null || true
```

也可以检查构建目录生成的设备树：

```bash
find build_dir -name '*cudy-tr3000-mod*.dtb' -print
```

使用 `dtc` 反编译后，必须再次确认：

```text
compatible = "cudy,tr3000-mod", "mediatek,mt7981"
ubi reg = <0x5c0000 0x7000000>
```

## 12. 刷机前备份

先在 Linux 电脑创建备份目录：

```bash
mkdir -p "backup-tr3000-$(date +%F)"
cd "backup-tr3000-$(date +%F)"
```

记录路由器信息：

```bash
ssh root@10.0.0.1 'cat /proc/mtd' | tee proc-mtd.txt
ssh root@10.0.0.1 'ubus call system board' | tee system-board.json
ssh root@10.0.0.1 'opkg list-installed' | tee packages.txt
ssh root@10.0.0.1 'uci export' > uci-export.txt
```

逐分区备份：

```bash
for n in 0 1 2 3 4 5; do
  ssh root@10.0.0.1 "dd if=/dev/mtd${n} bs=128k 2>/dev/null" > "mtd${n}.bin"
done
sha256sum mtd*.bin | tee SHA256SUMS
```

检查大小：

```bash
ls -lh mtd*.bin
```

`Factory` 和 `bdinfo` 包含每台设备独有的校准与 MAC 数据，必须至少保存两份，并放在不同存储设备上。备份文件不得提交到公开 Git 仓库。

## 13. 安全升级路径

### 13.1 从当前 OpenWrt/LuCI 升级

在 LuCI 的“系统 → 备份/升级”上传：

```text
*cudy_tr3000-mod-squashfs-sysupgrade.bin
```

第一次跨固件升级建议不保留配置，以避免旧版 UCI 配置和新插件冲突。只有确认配置结构兼容后才保留配置。

升级前可先通过 SSH 做兼容性检查：

```bash
scp *cudy_tr3000-mod-squashfs-sysupgrade.bin root@10.0.0.1:/tmp/firmware.bin
ssh root@10.0.0.1 'sysupgrade -T /tmp/firmware.bin'
```

`sysupgrade -T` 成功只表示格式和设备元数据通过检查，不等于已经刷写。

### 13.2 从不死 U-Boot 网页恢复

1. 路由器断电。
2. 电脑通过网线连接路由器 LAN 口。
3. 电脑设置静态地址：

   ```text
   IP：192.168.1.2
   掩码：255.255.255.0
   网关：留空或 192.168.1.1
   ```

4. 按住 Reset 后通电，等待红灯进入恢复状态。
5. 浏览器打开 `http://192.168.1.1`。
6. 确认页面显示 `Current mtd layout: 112m`。
7. 选择 `112m` 布局。
8. 上传 `cudy_tr3000-mod-squashfs-sysupgrade.bin`。
9. 等待写入和自动重启，中途不得断电、拔线或关闭页面。
10. 恢复电脑 DHCP，访问固件默认管理地址。

在没有再次确认文件名、哈希、目标设备和 `112m` 布局前，不要点击最终刷写按钮。

## 14. 首次启动验证清单

刷机后依次检查：

```text
[ ] 能进入 LuCI
[ ] board_name 为 cudy,tr3000-mod
[ ] /proc/mtd 中 ubi 起始和大小正确
[ ] 2.4 GHz Wi-Fi 可用
[ ] 5 GHz Wi-Fi 可用
[ ] WAN/LAN 口方向正确
[ ] 2.5G 网口正常协商
[ ] MAC 地址与刷机前一致
[ ] USB 3.0 正常识别
[ ] 重启后配置能够保存
[ ] U-Boot 恢复页面仍可进入
[ ] fwxd、rpcd、firewall4 无循环崩溃
[ ] 温度、CPU、内存和存储数据正常
```

SSH 检查命令：

```bash
ubus call system board
cat /tmp/sysinfo/board_name
cat /proc/mtd
ip -br link
iwinfo
logread | grep -Ei 'error|fail|fwx|ubi|nand'
dmesg | grep -Ei 'error|fail|ubi|nand|mt7981'
df -h
free -h
```

## 15. 建议的 AI 工作流程

让 AI 按以下阶段执行，每阶段结束后报告结果：

1. **只读审计**：检查分支、工作树、设备定义和分区参数。
2. **环境检查**：检查依赖、磁盘和非 root 用户，不立即编译。
3. **feeds 检查**：安装固定版本 feeds，列出可用 FanchmWrt/LuCI 插件。
4. **配置设计**：生成精简版 `.config`，解释每个新增插件的用途和依赖。
5. **配置验证**：运行 `make defconfig`，确认只有 `cudy_tr3000-mod` 被选中。
6. **下载阶段**：执行 `make download` 并报告下载失败项。
7. **编译阶段**：并行编译，失败后使用 `-j1 V=s` 定位。
8. **产物审计**：验证文件类型、设备元数据、DTB 分区和 SHA-256。
9. **停止等待**：不得自动上传或刷机，由用户检查产物和备份。

可直接交给 AI 的最终任务示例：

```text
请按照 TR3000_AI_BUILD_GUIDE.md 为 cudy_tr3000-mod 准备并编译固件。

先只读检查仓库和 Linux 环境，报告：
1. 当前分支和 git diff；
2. 112M DTS 与 filogic.mk 是否正确；
3. 缺失的编译依赖；
4. feeds 中可用的 FanchmWrt/LuCI 插件；
5. 建议的固件包列表和预计风险。

得到这些结果后继续完成配置、下载和编译。最终只输出经过校验的
cudy_tr3000-mod-squashfs-sysupgrade.bin 路径、大小、SHA-256 和构建日志摘要。
不要连接或刷写路由器，不要生成或建议刷写 BL2、FIP、preloader 和 sysupgrade.itb。
```

## 16. 回退原则

出现以下任一情况时停止刷写：

- 产物名称不是 `cudy_tr3000-mod`。
- 产物后缀是 `.itb`。
- DTB 中 UBI 不是 `0x5c0000/0x7000000`。
- `sysupgrade -T` 检查失败。
- Factory 或 bdinfo 没有完成双份备份。
- U-Boot 恢复页面不能进入。
- 恢复页面没有显示 112m 布局。
- 编译日志中仍存在镜像过大、UBI、DTB 或打包错误。

保留当前不死 U-Boot、完整 MTD 备份和一份已知可启动的 112M 固件，是本项目可恢复性的基础。


