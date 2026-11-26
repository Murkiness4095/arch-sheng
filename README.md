# Arch Sheng

小米平板 6S Pro（`sheng`）的 Arch Linux ARM 移植。使用 Arch Linux ARM 软件源和 `pacstrap` 生成 aarch64 基础系统，使用 `nixos-sheng` 的 U-Boot 从 ext4 分区启动。首版提供 TTY 和 NetworkManager，桌面会话可按需安装。

之前的 Omarchy 版已保留在 [omarchy-archive](https://github.com/Murkiness4095/arch-sheng/tree/omarchy-archive) 标签。已有 Omarchy 安装要换成 Arch Linux ARM，需要按新系统重新安装。

## GitHub Actions

推送到 `main` 会触发两条独立的 ARM64 构建，也可分别手动运行。默认包含设备固件。

| 工作流 | 根分区 | 镜像产物 |
| --- | --- | --- |
| [单系统](https://github.com/Murkiness4095/arch-sheng/actions/workflows/build-single.yml) | `userdata` | `rootfs-userdata`、`uboot-userdata` |
| [双系统](https://github.com/Murkiness4095/arch-sheng/actions/workflows/build-dual.yml) | `linux` | `rootfs-linux`、`uboot-linux` |

镜像带 SHA-256 校验文件。双系统布局参考 [nixos-sheng 的 Android A 槽 / Linux B 槽](https://github.com/Murkiness4095/nixos-sheng)，已有 Linux 系统如果占用同一个分区，刷写会覆盖它。首次启动准备 USB 键盘，在 TTY1 设置 root 密码，然后登录。联网可用 `nmtui`。

## 内核滚动更新

Arch Linux ARM 软件包和设备内核使用同一个 `sudo pacman -Syu` 更新。安装新版 `linux-sheng` 后，pacman 钩子部署 Image、DTB 和模块，构建 initramfs，并写入 extlinux 启动项。U-Boot 平板按键菜单提供旧内核入口。确认新内核可用后执行 `sudo sheng-boot confirm`，启动项生成失败可用 `sudo sheng-boot install` 重试。

每次 CI 都生成 `sheng-pacman-repository` 产物。没有配置长期软件源时，镜像内置本次构建的签名仓库，只适合首启测试。要让已安装系统持续收到设备更新，在 GitHub 仓库设置长期使用的 `SHENG_SIGNING_KEY` secret 和 `SHENG_REPO_URL` variable，并把同一构建的仓库文件部署到该 HTTPS 地址。后续版本使用同一签名密钥，内核有变更时递增 `pkgver` 或 `pkgrel`。部署时先发布新版包与签名，再切换仓库索引。设备发布地址需要和旧 Omarchy 仓库区分开。

首轮真机验收依次检查 TTY 登录和联网、`pacman -Syu` 安装递增版本的内核、重启核对 `uname -r`，再从 U-Boot 启动旧内核验证回退。回退入口只切换内核和它的启动文件。

设备适配参考 [debian-sheng](https://github.com/ianchb/debian-sheng)，U-Boot 构建复用 [nixos-sheng](https://github.com/Murkiness4095/nixos-sheng)。
