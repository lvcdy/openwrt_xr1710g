<img src="https://avatars.githubusercontent.com/u/53193414?s=200&v=4" alt="logo" width="200" height="200" align="right">

# OpenWrt XR1710G

这是一个面向 XR1710G 设备的 OpenWrt / ImmortalWrt 定制构建树。
仓库保留了上游 OpenWrt 的整体目录结构，并在 `package/emortal/` 下加入了本地定制包和 LuCI 功能。

## 仓库内容

- 基础 OpenWrt / ImmortalWrt 构建系统
- 本地定制包和 LuCI 应用：
  - `autosamba`
  - `default-settings`
  - `luci-app-adguardhome`
  - `luci-app-airoha-npu`
  - `luci-app-mlo`
  - `luci-app-netspeedtest`
  - `luci-app-w1700k-fancontrol`
  - `speedtest-go`
- 上游 OpenWrt 提供的目标平台、工具链和镜像构建脚本

## 构建方法

1. 克隆本仓库。
2. 执行 `./scripts/feeds update -a`。
3. 执行 `./scripts/feeds install -a`。
4. 执行 `make menuconfig` 进行配置。
5. 执行 `make -j$(nproc)` 开始编译。

## 推荐环境

- 推荐使用 Debian 11 或更高版本。
- 建议使用 x86_64 主机，至少 4 GB 内存和 25 GB 可用磁盘空间。
- 编译路径请保持为大小写敏感、无空格、无中文或其他非 ASCII 字符。
- 需要可用的网络连接以下载源码和依赖。

## 依赖说明

建议参考 OpenWrt 官方构建文档完成主机环境准备：

- [构建系统安装说明](https://openwrt.org/docs/guide-developer/build-system/install-buildsystem)
- [WSL 构建系统说明](https://openwrt.org/docs/guide-developer/build-system/wsl)

## 相关链接

- [OpenWrt](https://openwrt.org)
- [ImmortalWrt](https://github.com/immortalwrt/immortalwrt)
- [LuCI](https://github.com/immortalwrt/luci)

## GitHub Actions

仓库已提供一个手动触发的工作流，名称为“手动构建固件”。
进入 GitHub 的 `Actions` 页面后，选择该工作流并手动运行，即可按需构建不同设备的固件。

## 许可

本仓库遵循上游 OpenWrt / ImmortalWrt 的许可协议，具体请查看 `COPYING`。
