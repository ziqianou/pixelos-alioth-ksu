# PixelOS 17 alioth — KernelSU-Next + SUSFS kernel (GitHub Actions)

在 GitHub Actions 上为 **PixelOS 17 / alioth（Redmi K40 / POCO F3）** 编译内置
**KernelSU-Next（non-GKI manual hook）+ SUSFS v2.3.0** 的 4.19 内核，并重打包成可用
`fastboot` 刷入的 `boot.img`。

## 为什么用 LineageOS 24.0 内核

PixelOS 17 的内核源码仓库 `gitlab.com/PixelOS-alioth/kernel_xiaomi_alioth` 是**私有**的，
公开 CI 拉不到。因此本仓库使用公开的：

- 底座：`LineageOS/android_kernel_xiaomi_sm8250`，分支 `lineage-24.0`（同为 4.19.325 / Android 17 / alioth）
- 集成补丁：来自 `grill-glitch/kernel_xiaomi_sm8250` 的 KernelSU-Next + SUSFS 补丁
- KernelSU 源码：来自 `crdroidandroid/android_kernel_xiaomi_sm8150` 的 `drivers/kernelsu`（自带 SUSFS Kconfig）

为了不破坏 `/vendor/lib/modules`，构建时把 localversion 固定成设备当前内核的：

```
4.19.325-cip135-st19-perf-g2fbe35cee4e6
```

CI 中关闭 LTO/CFI（runner 磁盘/内存有限）；KSU/SUSFS 不依赖它们，vermagic 也不受影响。

## 目录

```
.github/workflows/build.yml      # 编译 + 重打包
patches/kernelsu-susfs.diff      # KernelSU-Next + SUSFS 内核集成补丁
patches/ksu-newfstat-hook.patch  # 让 fstat(2) 报告追加的 KSU init.rc 长度（Android 17 init）
config/device.config             # 从设备 /proc/config.gz 导出的完整配置
config/ksu.fragment              # KSU/SUSFS/vermagic/LTO 覆盖
ksu/drivers_kernelsu/            # KernelSU-Next + SUSFS 内核源码
```

## 使用

1. 新建一个 GitHub 仓库，把本目录推上去：

```sh
git init
git add .
git commit -m "PixelOS alioth KSU kernel build"
git remote add origin https://github.com/<you>/<repo>.git
git push -u origin main
```

2. 打开 **Actions → Build PixelOS alioth KernelSU kernel → Run workflow**
   - `rom_url`：和你手机上已安装的 PixelOS 版本对应（默认 2026-09-22 那版）
   - `kernel_ref`：默认固定到 `6b34d24765327d746074c8e53b5ad0d24a51acc7`
   - `jobs`：默认 4（GitHub 免费 runner 4 核）

3. 跑完后下载 artifact：`boot-ksu.img`、`Image`、`KernelSU_Next_v3.4.2_33355-release.apk`。

## 刷入（当前槽 `_b`）

先备份 stock boot（Magisk 已 root，可直接 dd）：

```sh
adb shell "su -c 'dd if=/dev/block/by-name/boot_b of=/data/local/tmp/boot_b.img'"
adb pull /data/local/tmp/boot_b.img
adb shell "su -c 'dd if=/dev/block/by-name/vbmeta_b of=/data/local/tmp/vbmeta_b.img'"
adb pull /data/local/tmp/vbmeta_b.img
```

先**临时启动**测试，不写任何分区：

```sh
adb reboot bootloader
fastboot boot boot-ksu.img
```

能正常开机后，再正式刷入：

```sh
fastboot flash boot_b boot-ksu.img
fastboot reboot
```

如果卡 fastboot / 不开机，恢复：

```sh
fastboot flash boot_b boot_b.img
fastboot reboot
```

首次开机后安装 `KernelSU_Next_v3.4.2_33355-release.apk`，授予 root，再按需安装
SUSFS 模块、Zygisk Next、LSPosed 等。

## OTA

内置 KSU 内核会被 PixelOS OTA 覆盖。更新后：

- 用新 ROM 的 zip URL 重新跑一次 workflow；
- OTA 装到 inactive slot 后先别重启，把新的 `boot-ksu.img` 刷到**重启后将使用的 slot**；
  或等 OTA 完成后再进 fastboot 刷 `boot_b`（注意当前 slot 可能变化）。

> 内置 KSU 无法使用 KernelSU 管理器的 "install to inactive slot" 功能。

## 已知风险

- 内核源码是 LineageOS 24.0，不是 PixelOS 私有源码。vermagic 已精确对齐，但
  `/vendor/lib/modules` 的 3 个模块（`rmnet_perf`、`rmnet_shs`、`gspca_main`）的符号 CRC
  仍可能不匹配而加载失败。它们不是开机必需。
- LTO/CFI 关闭，与原厂内核不同，只影响安全加固。
- 未在真机验证；务必先 `fastboot boot` 测试，确认能开机再写分区。
- 若 PixelOS 后续把内核升到 `cip136/st20`，需要更新 `kernel_ref` 并重新核对 vermagic。
