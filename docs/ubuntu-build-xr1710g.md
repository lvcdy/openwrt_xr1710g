# Ubuntu 上打包 XR1710G 镜像指南

本文面向在 Ubuntu 上本地编译本仓库固件的人，目标是生成 Airoha AN7581 平台下的 XR1710G 镜像。仓库当前的主目标是 `airoha/an7581`，其中 `gemtek_xr1710g-ubi` 是 XR1710G 的默认构建目标。

## 1. 推荐环境

- Ubuntu 22.04 或更高版本，Ubuntu 26.04 LTS 也可以直接使用。
- x86_64 主机。
- 至少 4 GB 内存，建议 8 GB 以上。
- 至少 25 GB 可用磁盘空间，实际建议预留 40 GB 以上。
- 编译目录不要放在带空格、中文或其他非 ASCII 字符的路径下。
- 需要稳定网络访问源码和下载包。

## 2. 安装系统依赖

先更新系统，再安装 OpenWrt 常用依赖。仓库的 GitHub Actions 里使用的是下面这一组包，适合作为 Ubuntu 本地编译的基线：

```bash
sudo apt-get update
sudo apt-get install -y \
  build-essential bison ccache clang curl fastjar flex gawk gettext \
  git gcc-multilib g++-multilib libelf-dev libncurses-dev libssl-dev \
  libpython3-dev patch pkg-config python3 python3-venv python3-pip \
  python3-setuptools qemu-utils rsync subversion swig unzip vim wget \
  xsltproc zlib1g-dev zstd

python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install pyelftools
```

如果系统提示缺少额外工具，再按报错补装即可。不同 Ubuntu 版本的包名可能略有差异，Ubuntu 26.04 LTS 上如果个别包名变化，按实际报错替换即可，但上面这组通常足够覆盖本仓库的本地编译。

已核对 Ubuntu 26.04 LTS 的包名可用性：`fastjar`、`build-essential`、`python3-pip`、`libssl-dev`、`libpython3-dev` 都存在；`libncurses5-dev` 与 `python3-distutils` 没有精确包名，前者应使用 `libncurses-dev`，后者从安装列表中移除即可。

Ubuntu 26.04 LTS 默认启用了 PEP 668 保护，不建议直接对系统 Python 执行 `pip install --upgrade pip`。本指南改用本地虚拟环境 `.venv` 来安装 `pyelftools`，这样不会触发 `externally-managed-environment` 错误。

## 3. 拉取仓库

```bash
git clone <你的仓库地址>
cd openwrt_xr1710g
```

如果你已经有源码目录，直接进入仓库根目录即可。

## 4. 修复脚本权限

在 Ubuntu 上首次使用前，建议补一下脚本执行权限，避免后续 `feeds`、`menuconfig` 或检查脚本报权限错误。这个步骤尤其重要，如果源码是从压缩包、Windows 共享目录或不保留执行位的方式拷贝过来的，脚本权限很容易丢失：

```bash
chmod +x config/check-*.sh scripts/feeds scripts/*.sh scripts/*.pl scripts/config.guess scripts/config/*.sh scripts/ipkg-* scripts/*-package.sh
```

如果你已经遇到过 `scripts/ipkg-remove: Permission denied`，通常就是这里漏掉了 `scripts/ipkg-*` 的执行位。补完之后再重新跑 `make` 即可。

## 5. 更新并安装 feeds

仓库的构建流程依赖 feeds 中的额外软件包。先更新，再安装：

```bash
perl ./scripts/feeds update -a
perl ./scripts/feeds install -a
```

如果你的 shell 环境里没有 `perl`，请先安装它。Ubuntu 通常默认会带。

## 6. 生成编译配置

这个仓库的 GitHub Actions 已经把 XR1710G 的默认配置写清楚了。对于本地 Ubuntu 编译，推荐直接按同样的目标生成配置，这样可以最大程度避免选错 target、subtarget 或 device profile。

XR1710G 对应的编译目标由三层组成：

- Target System: `Airoha`
- Subtarget: `an7581`
- Target Profile: `Gemtek XR1710G (UBI)`

如果你想手工检查，也可以先跑一次 `make menuconfig`，确认路径是否选对：

1. 运行 `make menuconfig`。
2. 进入 `Target System`，选择 `Airoha`。
3. 进入 `Subtarget`，选择 `an7581`。
4. 进入 `Target Profile`，选择 `Gemtek XR1710G (UBI)`。
5. 保存退出。

如果你想直接生成可复现的配置文件，推荐用下面这段命令。它会把 XR1710G 的目标写进 `.config`，再用 `make defconfig` 补齐依赖和默认项：

```bash
DEVICE=gemtek_xr1710g-ubi
DEVICE_CONFIG=${DEVICE//-/_}

printf '%s\n' \
  'CONFIG_TARGET_airoha=y' \
  'CONFIG_TARGET_airoha_an7581=y' \
  "CONFIG_TARGET_airoha_an7581_DEVICE_${DEVICE_CONFIG}=y" \
  'CONFIG_CCACHE=y' > .config

make defconfig
grep "^CONFIG_TARGET_airoha_an7581_DEVICE_${DEVICE_CONFIG}=" .config || true
```

这里每一项的作用是：

- `CONFIG_TARGET_airoha=y`：先选中 Airoha 这一整条目标树。
- `CONFIG_TARGET_airoha_an7581=y`：再把目标细化到 AN7581 子平台。
- `CONFIG_TARGET_airoha_an7581_DEVICE_gemtek_xr1710g_ubi=y`：最后锁定 XR1710G 的 UBI 设备配置。
- `CONFIG_CCACHE=y`：启用编译缓存，后续重复编译会快很多。

`make defconfig` 这一步不要省略。它会把仓库里其它必须的默认选项补出来，避免只写了核心目标却缺少依赖配置。

如果你想直接在图形界面里核对当前配置，`make menuconfig` 之后还可以用下面的命令确认最终选中的设备项是否真的生效：

```bash
grep '^CONFIG_TARGET_airoha_an7581_DEVICE_' .config
```

如果输出里出现的是 `CONFIG_TARGET_airoha_an7581_DEVICE_gemtek_xr1710g_ubi=y`，就说明配置已经对上了。

如果你需要别的设备，仓库当前还支持：

- `gemtek_w1700k-ubi`
- `gemtek_xg2010g-ubi`
- `nokia_valyrian`
- `nokia_xg-040g-md`
- `airoha_an7581-evb`
- `airoha_an7581-evb-emmc`

## 7. 先下载源码

正式编译前，先把源码包下载完整。这样可以把网络问题和编译问题拆开，排查更快：

```bash
make download -j"$(nproc)"
```

如果下载阶段只出现类似下面的警告：

- `python3-pysocks` 不存在
- `python3-unidecode` 不存在

这通常来自 `package/feeds/packages/onionshare-cli/Makefile` 的依赖声明，属于 feeds 里某些可选软件包的依赖不完整警告，不一定会阻止 `make download` 继续执行。如果你的目标并不包含 `onionshare-cli`，一般可以先忽略这些警告，继续观察下载是否真正失败。

如果下载阶段真的报错，先把下载错误解决，再继续编译。

## 8. 开始编译

本仓库建议直接并行编译：

```bash
make -j"$(nproc)"
```

如果你想要更稳妥的排错方式，首次构建也可以先单线程跑一遍：

```bash
make -j1 V=s
```

这样一旦失败，可以更容易定位到具体是哪一个镜像规则、哪一个文件缺失、或者哪一个命令返回了非零。

## 9. 产物在哪

成功后，XR1710G 的编译产物通常会出现在下面目录：

```bash
bin/targets/airoha/an7581/
```

对于 `gemtek_xr1710g-ubi`，重点关注这些类型的文件：

- `sysupgrade.itb`
- 可能还有与 U-Boot chainload 相关的 `itb` 产物

你可以用下面的命令快速查看：

```bash
find bin/targets/airoha/an7581 -maxdepth 2 -type f | sort
```

## 10. 这个仓库里 XR1710G 镜像是怎么拼出来的

理解这一点有助于排错。

仓库里 XR1710G 对应的设备定义位于 `target/linux/airoha/image/an7581.mk`，而 U-Boot 产物的安装规则位于 `package/boot/uboot-airoha/Makefile`。

构建链路大致是这样：

1. `uboot-airoha` 先把 `u-boot.bin`、`u-boot.bin.lzma`、`u-boot.dtb` 或 `u-boot.fip` 安装到 `STAGING_DIR_IMAGE`。
2. `target/linux/airoha/image/an7581.mk` 再从这些 staged 文件生成设备专用的 chainload 镜像。
3. 最终镜像写到 `bin/targets/airoha/an7581/`。

所以如果 `make` 在 `target/linux/install` 阶段失败，最常见原因通常是下面几类：

- 某个 staged 文件名对不上。
- U-Boot 变体没有生成对应的 `u-boot.bin` 或 `u-boot.dtb`。
- `mkits.sh` 或 `mkimage` 在拼 FIT 时失败。
- 目标配置选错，导致编译到了别的设备或别的 subtarget。

## 11. 当前版本状态

按当前仓库内容看，几个关键组件已经跟到比较新的版本线，不是陈旧冻结状态：

- 内核：`KERNEL_PATCHVER:=6.18`，测试内核为 `6.12`。
- hostapd：源码日期是 `2026-04-02`。
- iwinfo：源码日期是 `2026-01-14`，并且已带 `no-lto` 和多无线电兼容补丁。
- U-Boot：XR1710G 对应的 U-Boot 版本线是 `2026.01`。
- default-settings-chn：当前 release 是 `29`。

从仓库历史看，最近还在继续动的文件主要有两个：

- `target/linux/airoha/Makefile`：最近一次更新在 2026-05-11。
- `package/emortal/default-settings/Makefile`：最近一次更新在 2026-05-08。

这说明构建链路本身已经在往新版本走，当前更值得关注的不是“包太旧”，而是某些机型定制包是否真的适合 XR1710G，比如 `luci-app-w1700k-fancontrol` 这类沿用自 W1700K 的功能包。

## 12. 常见问题

### 12.1 `target/linux failed to build`

这表示失败点已经进入了目标镜像生成阶段，不是普通 package 编译。

建议按下面顺序排查：

1. 用 `make -j1 V=s target/linux/install` 重新跑一次。
2. 找到第一条真正的 error，而不是最后那句汇总错误。
3. 检查 `bin/targets/airoha/an7581/` 是否已有部分产物。
4. 检查 `STAGING_DIR_IMAGE` 里的 `an7581_*` 文件名是否存在。

### 12.2 依赖包警告

如果看到类似 `python3-pysocks` 或 `python3-unidecode` 不存在的警告，通常说明某个 feeds 软件包依赖还没完全对齐。这个警告不一定会立刻阻止 XR1710G 镜像构建，但最好在正式出镜前再确认一次 `feeds` 状态。

### 12.3 路径问题

如果源码放在 Windows 共享目录、含空格目录，或者含中文目录，OpenWrt 构建经常会出现奇怪的脚本问题。建议把源码放到纯英文路径下，例如：

```bash
/home/<用户名>/work/openwrt_xr1710g
```

### 12.4 `make menuconfig` 卡在 `scripts/config/mconf`

如果你在 Ubuntu 26.04 LTS 上运行 `make menuconfig`，却看到类似 `make -s -C scripts/config mconf: build failed` 的错误，通常是主机缺少 ncurses 开发库，或者 `pkg-config` 没有正确找到它。

优先检查并补装这两个包：

```bash
sudo apt-get update
sudo apt-get install -y libncurses-dev pkg-config
```

然后回到仓库根目录重新执行：

```bash
make menuconfig
```

仓库里的 `scripts/config/mconf-cfg.sh` 会优先用 `pkg-config` 查找 `ncursesw` 或 `ncurses`，找不到时才会去探测系统默认头文件路径。所以如果你已经装了 `libncurses-dev`，但仍然报错，下一步就应该检查 `pkg-config` 是否可用，以及 `/usr/include/ncursesw/` 下的头文件是否存在。

如果上面两个包已经安装，但 `make menuconfig` 还是失败，那就继续确认 ncurses 是否真的可被 `pkg-config` 解析：

```bash
pkg-config --exists ncursesw && echo ok
pkg-config --exists ncurses && echo ok
pkg-config --cflags ncursesw
pkg-config --libs ncursesw
```

如果这些命令里有任何一个失败，说明问题不在 OpenWrt 自身，而是在主机的 ncurses 发现链路上。此时建议再补装完整的主机编译工具：

```bash
sudo apt-get install -y build-essential bison flex gawk gettext
```

然后再次执行 `make menuconfig`。`mconf` 本身只是菜单前端，但它的编译和链接仍然依赖主机编译器、链接器和相关构建工具可用。

如果你是在补装依赖之后才重试 `make menuconfig`，但它还是报同样的错，再清理一次 `scripts/config` 的旧构建产物后重试：

```bash
make -C scripts/config clean
make menuconfig
```

这样可以排除 `mconf-cflags`、`mconf-libs` 或 `mconf-bin` 这类中间文件在前一次失败时被错误缓存的情况。

## 13. 推荐的一次性命令

如果你想从零开始一口气跑，可以在 Ubuntu 里执行：

```bash
sudo apt-get update
sudo apt-get install -y \
  build-essential bison ccache clang curl fastjar flex gawk gettext \
  git gcc-multilib g++-multilib libelf-dev libncurses-dev libssl-dev \
  libpython3-dev patch pkg-config python3 python3-venv python3-pip \
  python3-setuptools qemu-utils rsync subversion swig unzip vim wget \
  xsltproc zlib1g-dev zstd

python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install pyelftools

git clone <你的仓库地址>
cd openwrt_xr1710g

chmod +x config/check-*.sh scripts/feeds scripts/*.sh scripts/*.pl scripts/config.guess scripts/config/*.sh scripts/ipkg-* scripts/*-package.sh
perl ./scripts/feeds update -a
perl ./scripts/feeds install -a

DEVICE=gemtek_xr1710g-ubi
DEVICE_CONFIG=${DEVICE//-/_}
printf '%s\n' \
  'CONFIG_TARGET_airoha=y' \
  'CONFIG_TARGET_airoha_an7581=y' \
  "CONFIG_TARGET_airoha_an7581_DEVICE_${DEVICE_CONFIG}=y" \
  'CONFIG_CCACHE=y' > .config
make defconfig

make download -j"$(nproc)"
make -j"$(nproc)"
```

## 14. 参考文件

- [README.md](../README.md)
- [.github/workflows/manual-build.yml](../.github/workflows/manual-build.yml)
- [target/linux/airoha/an7581/target.mk](../target/linux/airoha/an7581/target.mk)
- [target/linux/airoha/image/an7581.mk](../target/linux/airoha/image/an7581.mk)
- [package/boot/uboot-airoha/Makefile](../package/boot/uboot-airoha/Makefile)
