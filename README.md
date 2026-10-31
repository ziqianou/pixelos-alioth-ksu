# PixelOS 17 alioth — KernelSU-Next kernel (GitHub Actions)

在 GitHub Actions 上为 **PixelOS 17 / alioth（Redmi K40 / POCO F3）** 编译内置
**KernelSU-Next（non-GKI manual hook）+ SUSFS** 的 4.19 内核，并重打包成可用
`fastboot` 刷入的 `boot.img`。

> 注意：内联的 SUSFS 是 **v1.5.2**。`ksu_susfs` 的 `set_uname` / `sus_path` /
> `sus_mount` 需要 v1.5.3+，在当前内核上会返回 `SUSFS operation not supported`；
> 目前只依赖 KSU 自带的 `kernel_umount` / `selinux_hide` / `avc_spoof`。

## 为什么用 LineageOS 24.0 内核

PixelOS 17 的内核源码仓库 `gitlab.com/PixelOS-alioth/kernel_xiaomi_alioth` 是**私有**的，
公开 CI 拉不到。因此本仓库使用公开的：

- 底座：`LineageOS/android_kernel_xiaomi_sm8250`，分支 `lineage-24.0`（同为 4.19.325 / Android 17 / alioth）
- 集成树：`grill-glitch/kernel_xiaomi_sm8250`（已带 KSU-Next/SUSFS 手动 hook 接入）
- KernelSU 源码：本仓库 `ksu/drivers_kernelsu/`（KernelSU-Next v3.4.0-legacy + SUSFS Kconfig/符号移植）

为了不破坏 `/vendor/lib/modules`，构建时把 localversion 固定成设备当前内核的：

```
4.19.325-cip135-st19-perf-g2fbe35cee4e6
```

CI 中关闭 LTO/CFI（runner 磁盘/内存有限）；KSU/SUSFS 不依赖它们，vermagic 也不受影响。

## 目录

```
.github/workflows/build.yml              # 编译 + 重打包
patches/ksu-newfstat-hook.patch          # fstat(2) 报告追加的 KSU init.rc 长度
patches/cap-checkpoint-restore.patch     # CAP_LAST_CAP 补到 40 + SELinux cap2 classmap
patches/memcg-lru-size-warn.patch        # memcg lru_size WARN 改为 pr_warn，避免 taint
patches/kernelsu-susfs.diff              # 历史参考，当前 workflow 不应用
config/device.config                     # 从设备 /proc/config.gz 导出的完整配置
config/ksu.fragment                      # KSU/SUSFS/vermagic/LTO 覆盖
ksu/drivers_kernelsu/                    # KernelSU-Next + SUSFS 内核源码
```

## 当前补丁

1. **`ksu-newfstat-hook.patch`**
   Android 17 libbase 按 `fstat(2)` 的 `st_size` 精确读取，不再读到 EOF。KSU 的 init.rc
   是追加在 EOF 的，所以给 `newfstat` 加上 `ksu_rc_len + module_rc_len`。

2. **`cap-checkpoint-restore.patch`**
   Android 17 native zygote（`/system/bin/zygote_next`）按完整 cap 列表 drop bounding
   set；补 `CAP_CHECKPOINT_RESTORE(40)`，并把 SELinux cap2 classmap 按
   `plat_sepolicy.cil` 的顺序补上 `checkpoint_restore`。

3. **`memcg-lru-size-warn.patch`**
   4.19 的 `mem_cgroup_update_lru_size()` 在内存压力下会触发 `WARN_ONCE`，给内核打上
   `TAINT_WARN`（`/proc/sys/kernel/tainted` 非 0）。改为 `pr_warn_once`，保留日志和
   `lru_size=0` 重置，但不再 taint。

4. **KSU 源码改动**（`ksu/drivers_kernelsu/`）
   - init.rc 改回 append（排在系统 post-fs-data 之后，等 `installkey /data` 完成）
   - sucompat / stat / faccessat 跳过 `_process_zygote` / `zygote-child`，避免
     `app_zygote -> ksu` 的 bounded transition 失败

## 构建

Workflow 会：

1. 浅克隆 `grill-glitch/kernel_xiaomi_sm8250` 到默认 `kernel_ref`
   `d736f87b1b7e07d1be1f73e94a2b208b6265274a`
2. 把 `ksu/drivers_kernelsu` 复制到 `kernel/drivers/kernelsu`
3. 应用上面三个 kernel patch
4. 用 stock PixelOS 内核的 UTS 版本字符串构建：
   `#1 SMP PREEMPT Tue Sep 22 17:03:44 CEST 2026`，`ci@PixelOS-CI`，
   `KBUILD_BUILD_VERSION=1`
5. 拉取 stock `boot.img`，把 `Image` 重打包成 `boot-ksu.img`
6. 下载 KernelSU-Next v3.4.0 manager（`KernelSU_Next_v3.4.0_33294-release.apk`）

## 使用

仓库：<https://github.com/ziqianou/pixelos-alioth-ksu>

打开 **Actions → Build PixelOS alioth KernelSU kernel → Run workflow**
- `rom_url`：默认 `PixelOS_alioth-17.0-20260922-1500.zip`
- `kernel_ref`：默认 `d736f87...`
- `jobs`：默认 4

跑完后下载 artifact：`boot-ksu.img`、`Image`、`System.map`、`.config`、
`KernelSU_Next_v3.4.0_33294-release.apk`。

## 刷入 / 回滚（当前槽 `_b`）

本机整合后的路径（仓库在 `repo/`，产物在 `../artifacts/`）：

```sh
# 最新已刷构建
fastboot flash boot_b ../artifacts/uname/pixelos-alioth-ksu-14/boot-ksu.img
fastboot reboot

# 回滚到 stock/Magisk 备份
fastboot flash boot_b ../artifacts/main/boot_b.backup.img
fastboot reboot

# 临时启动测试
fastboot boot ../artifacts/uname/pixelos-alioth-ksu-14/boot-ksu.img
```

首次开机后安装 `KernelSU_Next_v3.4.0_33294-release.apk`，授予 root，再按需安装
Zygisk Next、LSPosed、SUSFS 模块等。

## 本地工作区

本仓库所在的工作区还包含（不随 git 推送）：

```
../artifacts/          # 各次构建产物，main=回滚备份，uname=最新已刷
../cache/alioth-work/  # 本地内核源码/clang/工具/ROM dump
../downloads/          # ROM、boot、vbmeta、LSPosed、KernelSU APK 等
```

详见 `../README.md`。

## 设备侧状态 / 已知限制

- KSU built-in v3.4.0 + manager v3.4.0；Zygisk Next 1.5.2；LSPosed v2.2.1-log 7936；HMA
- `cap_last_cap=40`；`uname -v` 已对齐 stock；`tainted=4`（仅 `S`）
- SUSFS 1.5.2 的 userspace ABI 不支持 `set_uname` / `sus_path` / `sus_mount`
- `reqable-magisk` 已 disable（去掉 CA 和系统挂载，规避检测）；需要时
  `ksud module enable reqable-magisk` + 重启
- `untrusted_app` 读取 `/proc/version`、`/proc/sys/kernel/osrelease` 的 SELinux 规则
  在设备的 `/data/adb/modules/susfs4ksu/sepolicy.rule`（不在仓库）
- Hunter：87/97 通过；5 高风险（2×Kernel Runtime、Automation、Notification Bar、
  Network Trust）；Kernel Runtime 剩余原因是 `rawMode=libc_syscall_fallback`
- Play Integrity：BASIC 通过，DEVICE/STRONG 受 keybox/TEE 限制，未解决

## OTA

内置 KSU 内核会被 PixelOS OTA 覆盖。更新后：

- 用新 ROM 的 zip URL 重新跑一次 workflow；
- OTA 装到 inactive slot 后先别重启，把新的 `boot-ksu.img` 刷到**重启后将使用的 slot**；
  或等 OTA 完成后再进 fastboot 刷 `boot_b`（注意当前 slot 可能变化）。

> 内置 KSU 无法使用 KernelSU 管理器的 "install to inactive slot" 功能。

## 已知风险

- 内核源码是 LineageOS 24.0 + grill 树，不是 PixelOS 私有源码。vermagic 已精确对齐，
  但 `/vendor/lib/modules` 的 3 个模块（`rmnet_perf`、`rmnet_shs`、`gspca_main`）的
  符号 CRC 仍可能不匹配而加载失败。它们不是开机必需。
- LTO/CFI 关闭，与原厂内核不同，只影响安全加固。
- 已在真机 alioth / `boot_b` 上验证开机、KSU post-fs-data、Zygisk/LSPosed/HMA、
  Chrome/app zygote、Hunter 扫描。
- 若 PixelOS 后续把内核升到 `cip136/st20`，需要更新 `kernel_ref` 并重新核对 vermagic。
