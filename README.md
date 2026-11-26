# Arch Sheng

小米平板 6S Pro（`sheng`）的 Arch Linux ARM 移植。使用 Arch Linux ARM 软件源和 `pacstrap` 生成 aarch64 基础系统，使用 `nixos-sheng` 的 U-Boot 从 ext4 分区启动。镜像可选 TTY、GNOME、KDE Plasma、Niri 或 Denial，均使用 NetworkManager。

之前的 Omarchy 版已保留在 [omarchy-archive](https://github.com/Murkiness4095/arch-sheng/tree/omarchy-archive) 标签。已有 Omarchy 安装要换成 Arch Linux ARM，需要按新系统重新安装。

## GitHub Actions

推送到 `main` 会触发一条 ARM64 构建，先生成一次软件包，再并行组装单系统和双系统镜像。下列两个工作流也可独立手动运行。默认包含设备固件。

| 工作流 | 根分区 | 镜像产物 |
| --- | --- | --- |
| [单系统](https://github.com/Murkiness4095/arch-sheng/actions/workflows/build-single.yml) | `userdata` | `rootfs-userdata-<profile>`、`uboot-userdata` |
| [双系统](https://github.com/Murkiness4095/arch-sheng/actions/workflows/build-dual.yml) | `linux` | `rootfs-linux-<profile>`、`uboot-linux` |

镜像带 SHA-256 校验文件。双系统布局参考 [nixos-sheng 的 Android A 槽 / Linux B 槽](https://github.com/Murkiness4095/nixos-sheng)，已有 Linux 系统如果占用同一个分区，刷写会覆盖它。首次启动准备 USB 键盘，在 TTY1 创建可使用 sudo 的普通账户。桌面版随后进入图形登录。联网可用 `nmtui`。

## 内核滚动更新

Arch Linux ARM 软件包和设备内核使用同一个 `sudo pacman -Syu` 更新。安装新版 `linux-sheng` 后，pacman 钩子部署 Image、DTB 和模块，构建 initramfs，并写入 extlinux 启动项。U-Boot 平板按键菜单提供旧内核入口。确认新内核可用后执行 `sudo sheng-boot confirm`，启动项生成失败可用 `sudo sheng-boot install` 重试。

每次 CI 都生成 `sheng-pacman-repository` 产物。没有配置长期软件源时，镜像内置本次构建的签名仓库，只适合首启测试。要让已安装系统持续收到设备更新，在 GitHub 仓库设置长期使用的 `SHENG_SIGNING_KEY` secret 和 `SHENG_REPO_URL` variable，并把同一构建的仓库文件部署到该 HTTPS 地址。后续版本使用同一签名密钥，内核有变更时递增 `pkgver` 或 `pkgrel`。部署时先发布新版包与签名，再切换仓库索引。设备发布地址需要和旧 Omarchy 仓库区分开。

首轮真机验收依次检查 TTY 登录和联网、`pacman -Syu` 安装递增版本的内核、重启核对 `uname -r`，再从 U-Boot 启动旧内核验证回退。回退入口只切换内核和它的启动文件。

设备适配参考 [debian-sheng](https://github.com/ianchb/debian-sheng)，U-Boot 构建复用 [nixos-sheng](https://github.com/Murkiness4095/nixos-sheng)。

## 桌面配置

自动触发的构建使用 TTY。其余桌面从任一工作流的 **Run workflow** 菜单选择。

- Niri 镜像默认启动 Noctalia。Super+Space 打开启动器，Super+S 打开控制中心，Super+Enter 打开终端，Super+Q 关闭窗口，Super+O 打开概览。方向键用于切换窗口，按住 Shift 移动窗口，Page Up / Page Down 切换工作区。完整配置随用户模板安装到 `~/.config/niri/config.kdl`。
- Denial 使用自带的 Flutter 桌面和状态栏，随上游签名的 aarch64 软件源更新。Droidloom 建议等平板的 GPU 加速、Android 输入和休眠通过真机测试后按需安装。
- 如果使用长期设备软件仓库并发布 Niri 镜像，请用 Niri 构建的仓库产物，以便仓库同时包含 `noctalia`。

## 构建缓存

主分支推送构建只打包和签名一次，两个镜像任务共用同一份签名软件仓库。手动构建也会从 Actions 缓存复用未签名内核包、设备包、Noctalia 包和对应布局的 U-Boot。软件包在当前运行中重新签名，签名密钥和已签名索引不会进入缓存。

内核包缓存未命中时，构建仍会尝试复用 Linux 编译器缓存。软件包配方、构建脚本或固定构建容器有变化时会重新打包。遇到 Arch Linux ARM 上游工具链或库 ABI 变更时，可以更新仓库变量 `SHENG_PACKAGE_CACHE_EPOCH`，让下一次运行重建软件包与编译缓存。
