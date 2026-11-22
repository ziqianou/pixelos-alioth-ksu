# PixelOS alioth KernelSU 内核

给 Xiaomi Redmi K40 / POCO F3 / Mi 11X（`alioth`）的 **PixelOS 17（20261112）** 打一个
带 **KernelSU-Next + SUSFS** 的内核，并重打包成可直接 `fastboot flash boot` 的镜像。

构建全部在 **GitHub Actions** 完成，本地不需要 Android 构建环境。

- 仓库：`ziqianou/pixelos-alioth-ksu`
- ROM：`PixelOS_alioth-17.0-20261112-0826.zip`（`ro.custom.version` 为
  `PixelOS_alioth-17.0-20261112-0822`）
- 内核：`4.19.325-cip136-st20-perf-gf914d0b24613`
  `#1 SMP PREEMPT Thu Nov 12 09:32:12 CET 2026`
- Root：KernelSU-Next 3.4.2；隐藏：SUSFS v2.3.0
- 加固：LTO + CFI + SHADOW_CALL_STACK（与官方内核一致）

---

## 快速开始

### 1. 触发构建

```sh
gh workflow run build.yml --repo ziqianou/pixelos-alioth-ksu --ref main
gh run watch --repo ziqianou/pixelos-alioth-ksu
```

`workflow_dispatch` 的三个输入（一般不用改）：

| 输入 | 默认值 | 说明 |
| --- | --- | --- |
| `rom_url` | `…/PixelOS_alioth-17.0-20261112-0826.zip` | 用于提取 stock `boot.img` |
| `kernel_src_url` | `…/archive/f914d0b2461….tar.gz` | stock 内核源码 tarball |
| `jobs` | `4` | `make -j` 并行度 |

一次完整构建约 **24 分钟**（`Build Image` 约 14 分钟，ccache 命中约 45%）。

### 2. 下载产物

```sh
gh run download <run-id> --repo ziqianou/pixelos-alioth-ksu -n pixelos-alioth-ksu-<run_number>
```

| 文件 | 内容 |
| --- | --- |
| `boot-ksu.img` | 重打包好的镜像，直接刷入 |
| `kernel/out/arch/arm64/boot/Image` | 裸内核 |
| `kernel/out/System.map` | 符号表 |
| `kernel/out/.config` | 实际使用的配置 |
| `KernelSU_Next_v3.4.2_33355-release.apk` | 管理器 |
| `SHA256SUMS` | 校验 |

### 3. 刷入

```sh
adb reboot bootloader
fastboot flash boot_b boot-ksu.img     # 槽位按当前系统来，通常是 _b
fastboot --set-active=b                # 顺带复位 retry 计数，避免槽位降级
fastboot reboot
```

> ⚠️ 必须先 `fastboot --set-active=<slot>`，否则连续启动失败会把槽位标记成
> `unbootable`，bootloader 自动切到旧系统槽 → 无限重启（详见「排障」）。

> ⚠️ `fastboot boot <img>` 在这台机器上**是空操作**：返回 `Booting OKAY` 但不会加载
> 传入的镜像，只是重启进分区里的内核。只能用 `flash` 验证。

### 4. 验证

```sh
adb shell 'cat /proc/version'                         # 看编译器和版本串
adb shell 'su -v'                                     # 3.4.2:KernelSU
adb shell 'su -c "cat /proc/sys/kernel/tainted"'      # 0
adb shell 'su -c "stat -f /system | sed -n 2p"'       # 不是 overlay
adb shell 'su -c "cat /data/adb/modules/susfs4ksu/module.prop" | head -3'
```

---

## 构建是怎么做的

### 源码与二进制对齐

官方 20261112 的内核源码就是公开的 `PixelOS-Devices/android_kernel_xiaomi_sm8250`。
仓库固定到与 ROM 对应的提交：

```
f914d0b2461384fb80680ca7ebc8ec54df0a28a1
```

三层对齐决定了 `/vendor/lib/modules` 能不能加载：

| 层 | 做法 |
| --- | --- |
| vermagic | 覆盖 `localversion-cip/-st` 为 `-cip136`/`-st20`，并在 fragment 里固定 `CONFIG_LOCALVERSION="-perf-gf914d0b24613"`、关掉 `LOCALVERSION_AUTO`（tarball 没有 `.git`，否则会自动追加 `+`） |
| 配置 | 直接使用从 shipped `boot.img` 用 `scripts/extract-ikconfig` 提取的 `config/device.config` |
| 工具链 | AOSP prebuilt **`clang-r547379`（clang 20.0.0）**，与维护者 `build.sh` 一致 |

`build.sh` 里的 defconfig 链（`vendor/kona-perf_defconfig` + `vendor/xiaomi/*.config`）
**不能照抄**：它打开的 `CONFIG_CRYPTO_LZ4HC=y` 在 shipped 内核里是关闭的，而且这个符号在
4.19 这棵树上编不过（`crypto/lz4hc.c` 引用上游名 `LZ4HC_DEFAULT_CLEVEL`，
`lib/lz4/lz4hc.h` 只定义了 `LZ4HC_CLEVEL_DEFAULT`）。

### 配置

`config/device.config`（shipped 配置）+ `config/ksu.fragment`（叠加）：

```conf
CONFIG_KSU=y
CONFIG_KSU_MANUAL_HOOK=y
CONFIG_KSU_SUSFS=y
CONFIG_LOCALVERSION="-perf-gf914d0b24613"
# CONFIG_LOCALVERSION_AUTO is not set
CONFIG_KSU_SUSFS_SUS_OVERLAYFS=y     # 让 overlay 挂载对非 root 呈现底层 fs
```

LTO / CFI / SHADOW_CALL_STACK / `FRAME_WARN=4096` 全部沿用 shipped 值。

### 补丁

| 文件 | 作用 |
| --- | --- |
| `patches/kernelsu-susfs.diff` | KSU-Next 手动 hook 总集成（`fs/susfs.c`、`include/linux/susfs*.h`、execve/open/stat/faccessat/reboot 等 hook） |
| `patches/ksu-newfstat-hook.patch` | Android 17 的 libbase 按 `fstat(2)` 的 `st_size` 精确读取，需让 KSU 追加的 init.rc 长度可见 |
| `patches/memcg-lru-size-warn.patch` | 4.19 的 `lru_size` 调试检查改成 `pr_warn`，避免每次开机 taint（bit 9） |
| `patches/ksu-avc-spoof-hook.patch` | 让 `slow_avc_audit` 把 tsid 传给 KSU，`avc_spoof` 才生效 |
| `patches/no-cpu-taint.patch` | Qualcomm 的 big.LITTLE 特性表每次开机都会打 `TAINT_CPU_OUT_OF_SPEC`（`tainted=4`）并在 dmesg 留 warning；补丁去掉这两者 |
| `patches/cap-checkpoint-restore.patch` | 历史补丁，stock 源码已自带，**workflow 不应用** |

`SUS_OVERLAYFS` 的 Kconfig 定义在 `ksu/drivers_kernelsu/Kconfig`（上游整合 patch 里只引用了
`#ifdef`，没有 config 项，不补定义的话 `=y` 会被 `olddefconfig` 静默丢弃）。

### 流水线

1. 关闭多余磁盘占用、装构建依赖；
2. 下载 AOSP `clang-r547379` 并加进 `PATH`；
3. 下载源码 tarball（剥掉顶层目录）；
4. 拷入 `ksu/drivers_kernelsu`，打全部补丁；
5. 合并 `device.config` + `ksu.fragment`，`olddefconfig`，校验关键开关；
6. `make LLVM=1 LLVM_IAS=1 Image`，并固定 `KBUILD_BUILD_*` 使 `uname -v` 与 stock 一致；
7. 从 ROM 里抽 `boot.img`（`payload-dumper-go`），用 `mkbootimg` 换掉 kernel 重打包；
8. 上传 `boot-ksu.img`、`Image`、`System.map`、`.config`、KSU 管理器、校验和。

---

## 仓库结构

```
.github/workflows/build.yml     # 构建 + 重打包
config/device.config            # shipped 内核配置（extract-ikconfig）
config/ksu.fragment             # KSU / vermagic / SUSFS 覆盖
ksu/drivers_kernelsu/           # KernelSU-Next 3.4.2 legacy 源码
patches/                        # 见上表
README.md
```

---

## 已知限制

### `/vendor/lib/modules` 里三个模块加载失败

```
rmnet_perf: disagrees about version of symbol module_layout
rmnet_shs:  disagrees about version of symbol module_layout
gspca_main: disagrees about version of symbol module_layout
```

`module_layout` 的 CRC 由 `struct module` 算出，不匹配说明本仓库用的 `f914d0b` 与出包源码
仍有细微差异（已知 `techpack/audio` 有 3 个文件不同）。实测**不影响开机、通话和数据网络**，
只影响网络性能模块与 UVC 摄像头驱动。

### 反 root 检测的残留项

用 Hunter 6.72 扫描（114 项）仍有三类不计入「已修复」：

| 项 | 原因 | 可否修 |
| --- | --- | --- |
| `SELinux Policy Oracle Untrusted` | KSU 改写内核 sepolicy，多探针通道上下文不一致 | ❌ SUSFS 不参与策略生成，换版本无解 |
| `Find Prop Modify Mark`（`ro.boot.verifiedbootstate`） | Play Integrity 需要 `green`，只能 init 之后 `resetprop` | ❌ 固有取舍 |
| `Kernel Runtime Consistency`（同上根因） | prop 是 `green`、`/proc/cmdline` 是 `orange` | ❌ 同上 |

已修复并验证：`Kernel Tainted`（`tainted=0`）、`OVERLAYFS_SUPER_MAGIC`（`statfs /system`
返回 `ext3/4`，与隐藏后的 mountinfo 一致）、多余的 `ro.build.tags`/`ro.build.type` 写入。

### Play Integrity 只有 BASIC

Tricky Store 已装好、keybox 注入成功（KeyAttestation 能看到
`Key Attestation CA1 → Droid CA2 → Droid CA3 → TEE/bb0d83fa…`，并显示 bootloader 已上锁），
但 Google 只给 `MEETS_BASIC_INTEGRITY`——公开群发放的 keybox 被静默降级的典型表现。
当前 keybox 叶证书 **2026-12-01 到期**。

### 蓝牙 A2DP 无声音（ROM 侧问题）

20261112 上 A2DP 播放会让 ADSP 崩（`err_qdi.c:1038:EX:audio_process`，ADSP 每 ~1.8s 重启），
**stock 内核同样复现**，与本仓库内核无关。已报给维护者，等 ROM 修复。
临时方案：`scrcpy --audio-codec=opus --audio-source=playback`。

---

## 排障

### 卡第一屏 / 无限重启

按顺序排查：

```sh
fastboot getvar current-slot
fastboot getvar slot-unbootable:b
fastboot getvar slot-retry-count:b
```

槽位被标记不可启动时会自动切到另一个槽；如果那个槽是旧系统，它撑不起新的 `/data`，
表现就是「第一屏 → 约 20 秒 → 黑屏重启」的循环，此时**刷什么内核都没用**。恢复：

```sh
fastboot --set-active=b
fastboot flash boot_b <已知可启动的 boot.img>
fastboot reboot
```

### `Automatic_brick_rescue` 会放大故障

该模块（搞机助手生成）在 `post-fs-data` 里计数启动次数：等于 2 时
`chmod 000` 掉 `post-fs-data.d`/`service.d` 并禁用所有模块 + `reboot`；**≥4 时每次开机无条件
`reboot`**。一次超时启动就能把设备推进无限重启，而且 `chmod 000` 会顺带废掉 Zygisk。

排查内核问题时先禁用，事后建议直接卸载：

```sh
su -c 'touch /data/adb/modules/Automatic_brick_rescue/disable'
su -c 'echo 0 > /data/adb/modules/Automatic_brick_rescue/Number_of_starts.log'
```

### Zygisk 注入失败 / LSPosed 失效

症状：`/data/adb/zygisksu/.abort_msg` 是 `❌ Last injection failed!` 或
`❌ Module files corrupted`，没有 `zn-*` 进程，`znctl status` 报
`failed to connect to server`。

先查脚本权限（`chmod 000` 的后遗症），再考虑重装：

```sh
su -c 'ls -l /data/adb/post-fs-data.d /data/adb/service.d'
su -c 'chmod 755 /data/adb/post-fs-data.d/*.sh /data/adb/service.d/*.sh'
su -c 'mv /data/adb/modules/zygisksu/disable /data/local/tmp/ 2>/dev/null; reboot'
```

Tricky Store 若报 `verify1 failed / unverified!`，用同版本官方包
`ksud module install Tricky-Store-v*.zip` 干净重装即可。

### 不要在内核里改 `androidboot.verifiedbootstate`

试过在 `arch/arm64/kernel/setup.c` 里改写 `boot_command_line` 与
`/chosen/bootargs`（让 init 原生设置 `ro.boot.*`），结果**卡第一屏**。这些参数会让 init
决定是否走 locked/AVB 校验流程，太早改成 locked 就会校验失败。

### 其它踩过的坑

| 现象 | 原因 |
| --- | --- |
| `fastboot boot` 返回 OKAY 但版本没变 | 该机型 `fastboot boot` 是空操作 |
| SUSFS 的 `add_sus_mount` 报 not supported | 该命令在 v2.3.0 已废弃；`AUTO_ADD_SUS_*` 选项在此 patch 里只有声明没有实现，打开会链接失败 |
| `ksu_susfs enable_avc_log_spoofing` 无效 | 该实现只隐藏 KSU 域的 AVC 日志，不影响检测器的策略探针 |
| 想把 SUSFS 升级到更新版本 | 官方 4.19 分支停在 **v1.5.5**（2025-02）；本仓库的 v2.3.0 已经更新 |
| LTO 链接期 `lz4` 栈帧 warning | `FRAME_WARN=4096` 在 LTO 下由 `ld.lld` 报出，不吃 `-Werror`，官方内核同样有 |

---

## OTA 之后

PixelOS 更新会覆盖 `boot` 分区，需要重新构建并刷入：

1. 用新 ROM 的 zip URL 重新跑一次 workflow（必要时同步更新 `kernel_src_url` 到新提交）；
2. 等 OTA 落到 inactive slot 后，把新的 `boot-ksu.img` 刷到**重启后将使用的 slot**；
3. 如果版本串（`-cipXXX/-stXX`）变了，要同步改 `config/ksu.fragment` 和 workflow 里的
   `localversion-*`，否则 `/vendor/lib/modules` 不会加载。

> KernelSU 管理器的 "install to inactive slot" 对内置 KSU 无效。

---

## 参考与致谢

- 内核源码：[PixelOS-Devices/android_kernel_xiaomi_sm8250](https://github.com/PixelOS-Devices/android_kernel_xiaomi_sm8250)
- Root：[KernelSU-Next](https://github.com/KernelSU-Next/KernelSU-Next)
- 隐藏：[SUSFS](https://gitlab.com/simonpunk/susfs4ksu) / [susfs4ksu 模块](https://github.com/sidex15/ksu_module_susfs)
- 重打包：[payload-dumper-go](https://github.com/ssut/payload-dumper-go)、[mkbootimg](https://android.googlesource.com/platform/system/tools/mkbootimg)
- ROM：[PixelOS](https://pixelos.net/)（alioth 维护者 Tkiliay）
