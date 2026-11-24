# Omarchy Sheng

小米平板 6S Pro（`sheng`）的 Omarchy ARM64 移植。镜像基于官方 Omarchy ARM64 ISO 内的系统根镜像。桌面、账户初始化和用户空间升级沿用 Omarchy，设备内核由 `sheng` pacman 仓库提供。

## GitHub Actions 构建

[Build Omarchy Sheng](https://github.com/Murkiness4095/omarchy-sheng/actions/workflows/build.yml) 在 GitHub 的 ARM64 runner 上构建软件包、U-Boot 和 ext4 根镜像。推送默认构建双系统布局。手动运行工作流时可选择 `linux`（独立 Linux 分区）或 `userdata`（单系统）。下列产物以 `linux` 为例。

| 产物 | 内容 |
| --- | --- |
| `rootfs-linux` | `rootfs-linux.img.zst` 和 SHA-256 校验 |
| `uboot-linux` | 匹配 `linux` 分区的 Android 格式 U-Boot 镜像和 SHA-256 校验 |
| `sheng-pacman-repository` | 已签名的 aarch64 软件包、仓库数据库和公钥 |

工作流中的 `include_firmware` 选项用于构建包含设备固件的测试镜像。默认镜像省略设备固件覆盖包，可用 `packages/sheng-firmware` 在 Arch Linux ARM 上本地构建并安装。首次启动请准备实体键盘，用官方 Omarchy 的账户向导创建用户。

## 滚动更新

预览构建将该次构建的已签名仓库放入镜像。要使已有安装持续收到后续内核包，给 GitHub 仓库配置长期使用的 `SHENG_SIGNING_KEY` secret（专用于签包的无口令私钥），以及 `SHENG_REPO_URL` variable。主分支构建会用该密钥签名，并将该 URL 写入镜像。将 `sheng-pacman-repository` 产物发布到该 HTTPS 端点。对后续构建使用同一签名密钥，每次内核或配置变化都要递增 `pkgver` 或 `pkgrel`。上传新版包和签名后再切换仓库索引。

Omarchy 的 `omarchy update` 通过 pacman 安装 `linux-sheng`。ALPM 钩子为这个版本安装模块、生成 initramfs，并切换 extlinux 启动项。下次重启可用平板按键选择当前内核或上一套内核。确认图形、触摸和网络后运行 `sudo sheng-boot confirm`。如果启动项生成失败，可运行 `sudo sheng-boot install` 重试。该机制回退的是内核和启动文件，用户空间仍是 pacman 已升级的状态。

Omarchy 刷新渠道配置时会调用 `pre-refresh-pacman`。镜像为首个用户预置钩子，恢复 `sheng` 的仓库配置。手工替换 `/etc/pacman.conf` 后，可用 `sudo sheng-repo` 恢复。

## 真机验证

`linux` 镜像对应 [nixos-sheng 的 Android A 槽 / Linux B 槽布局](https://github.com/Murkiness4095/nixos-sheng)。`userdata` 镜像会覆盖原有用户数据。首次验证建议先用已有的 `linux` 测试分区和匹配的 U-Boot，确认首启进入 Omarchy 桌面。再发布一个递增版本的内核包，运行 `omarchy update`、重启并用 `uname -r` 核对。最后从 U-Boot 选择旧内核，核对版本、模块和外设。项目尚需完成上述真机验收。

设备参考 [debian-sheng](https://github.com/ianchb/debian-sheng)，U-Boot 构建复用 [nixos-sheng](https://github.com/Murkiness4095/nixos-sheng) 的固定提交。
