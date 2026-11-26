# Arch Sheng

小米平板 6S Pro（`sheng`）的 Arch Linux ARM 移植。使用 Arch Linux ARM 软件源和 `pacstrap` 生成 aarch64 基础系统，使用 `nixos-sheng` 的 U-Boot 从 ext4 分区启动。镜像可选 TTY、GNOME、Plasma Desktop、Plasma Mobile、Niri、Denial Desktop 或 Denial Mobile，均使用 NetworkManager。

之前的 Omarchy 版已保留在 [omarchy-archive](https://github.com/Murkiness4095/arch-sheng/tree/omarchy-archive) 标签。已有 Omarchy 安装要换成 Arch Linux ARM，需要按新系统重新安装。

## GitHub Actions

推送到 `main` 会由单系统工作流触发一条 ARM64 构建，先生成一次软件包，再并行组装单系统和双系统镜像。下列两个工作流也可独立手动运行。默认包含设备固件。

| 工作流 | 根分区 | 镜像产物 |
| --- | --- | --- |
| [单系统](https://github.com/Murkiness4095/arch-sheng/actions/workflows/build-single.yml) | `userdata` | `flash-userdata-<profile>` |
| [双系统](https://github.com/Murkiness4095/arch-sheng/actions/workflows/build-dual.yml) | `linux` | `flash-linux-<profile>` |

下载对应的 `flash-*` ZIP，解压一次即可得到可直接刷写的 Android sparse rootfs `.img`、配套 U-Boot `.img` 和 `SHA256SUMS`。刷机只需下载这个 ZIP。双系统布局参考 [nixos-sheng 的 Android A 槽 / Linux B 槽](https://github.com/Murkiness4095/nixos-sheng)，已有 Linux 系统如果占用同一个分区，刷写会覆盖它。镜像在构建时创建普通用户，默认用户名和密码都是 `sheng`。手动构建可以直接填写 `username` 和 `password`，留空使用默认值。图形镜像首次启动会进入图形登录。填写的密码会出现在 Actions 运行参数中。联网可用 `nmtui`。

## 内核滚动更新

Arch Linux ARM 软件包和设备内核使用同一个 `sudo pacman -Syu` 更新。安装新版 `linux-sheng` 后，pacman 钩子部署 Image、DTB 和模块，构建 initramfs，并写入 extlinux 启动项。U-Boot 平板按键菜单提供旧内核入口。确认新内核可用后执行 `sudo sheng-boot confirm`，启动项生成失败可用 `sudo sheng-boot install` 重试。

主分支构建使用长期密钥签名，并把 `sheng-pacman-repository` 发布到 [Arch Sheng 软件源](https://murkiness4095.github.io/arch-sheng/sheng.db)。新镜像预装公钥和软件源地址。公钥指纹为 `A65F04B9A574102C80A6419C2514325F46504FB2`。内核有变更时递增 `pkgver` 或 `pkgrel`，平板即可通过 `sudo pacman -Syu` 收到新版。

Fork 可通过 GitHub Actions Secrets 设置自己的 `SHENG_SIGNING_KEY`，通过 Variables 设置 `SHENG_REPO_URL`。如果用 GitHub Pages 托管，选择 GitHub Actions 作为发布来源，再设置 `SHENG_PUBLISH_PAGES=true`。没有远程软件源配置的构建使用镜像内置的测试仓库。

首轮真机验收依次检查 TTY 登录和联网、`pacman -Syu` 安装递增版本的内核、重启核对 `uname -r`，再从 U-Boot 启动旧内核验证回退。回退入口只切换内核和它的启动文件。

设备适配参考 [debian-sheng](https://github.com/ianchb/debian-sheng)，U-Boot 构建复用 [nixos-sheng](https://github.com/Murkiness4095/nixos-sheng)。

所有镜像均预装 Fastfetch。登录后运行 `fastfetch` 即可查看系统信息。

## 桌面配置

自动触发的构建使用 TTY。手动运行时，`profile` 提供 TTY、GNOME、KDE、Niri 和 Denial。KDE 可勾选 `plasma_mobile`，Denial 的 `denial_mobile` 默认勾选，取消即构建桌面模式。两个开关只对各自的桌面有效。

- Niri 镜像默认启动 Noctalia。Super+Space 打开 Noctalia 启动器，Super+D 打开 Fuzzel，Super+S 打开控制中心，Super+Enter 打开终端，Super+Q 关闭窗口，Super+O 打开概览。方向键用于切换窗口，按住 Shift 移动窗口，Page Up / Page Down 切换工作区。完整配置随用户模板安装到 `~/.config/niri/config.kdl`。
- `kde` 安装 Plasma Desktop。`plasma-mobile` 从同一个 Arch Linux ARM 软件源安装触屏 shell 和屏幕键盘，登录时预选 Plasma Mobile。两者各自构建镜像，共用 KWin、Plasma Workspace 和设备软件包。
- `denial` 使用 Denial 自带的 Flutter 桌面和状态栏。`denial-mobile` 使用同一个 aarch64 软件包，默认启用移动 shell、状态栏和屏幕键盘。两种模式都通过上游签名软件源更新。Droidloom 可按需安装。
- 图形镜像预装 Firefox、Fcitx5、Rime 朙月拼音和 CJK 字体，用 Ctrl+Space 切换中英文。GNOME 继续使用原生 Wayland 文本输入，GNOME Shell 内的候选窗可能需要 Kimpanel 扩展。移动版保留会话自带的屏幕键盘。
- 主分支发布的长期软件源也包含 Niri 镜像所用的 `noctalia`。

## 构建缓存

主分支推送构建只打包和签名一次，两个镜像任务共用同一份签名软件仓库。手动构建也会从 Actions 缓存复用未签名内核包、设备包、Noctalia 包和对应布局的 U-Boot。软件包在当前运行中重新签名，签名密钥和已签名索引不会进入缓存。

内核包缓存未命中时，构建仍会尝试复用 Linux 编译器缓存。软件包配方、构建脚本或固定构建容器有变化时会重新打包。遇到 Arch Linux ARM 上游工具链或库 ABI 变更时，可以更新仓库变量 `SHENG_PACKAGE_CACHE_EPOCH`，让下一次运行重建软件包与编译缓存。
