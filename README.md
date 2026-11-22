# PixelOS 17 alioth — KernelSU-Next kernel (GitHub Actions)

在 GitHub Actions 上为 **PixelOS 17 / alioth（Redmi K40 / POCO F3）** 编译内置
**KernelSU-Next（non-GKI manual hook）+ SUSFS** 的 4.19 内核，并重打包成可用
`fastboot` 刷入的 `boot.img`。

> SUSFS 内核代码是 **v2.3.0**，已接入 KSU 的 `reboot(0xDEADBEEF, 0xFAFAFAFA, cmd, arg)`
> 命令入口：`ksud susfs version/features` 与 `ksu_susfs` 的 `set_uname` / `add_sus_path` /
> `hide_sus_mnts_for_non_su_procs` / `enable_log` 等均可正常返回。
> 旧版已废弃命令（`add_sus_mount`、`sus_su`、`umount_for_zygote_iso_service`）会被忽略。

## 内核来源

底座：`github.com/PixelOS-Devices/android_kernel_xiaomi_sm8250`（公开），固定到
20261112 ROM 所用 stock 内核的提交
`f914d0b2461384fb80680ca7ebc8ec54df0a28a1`（4.19.325 / cip136 / st20，含
`techpack/audio`）。它与 ROM 的 ADSP 固件/音频 ABI 匹配。旧的
`grill-glitch@d736f87` 快照只对齐了 UTS 串，A2DP offload 会让 ADSP 固件
`err_qdi.c:1038 EX:audio_process` 崩溃（蓝牙无声），已弃用。workflow 直接下载该
提交的 GitHub archive。
- KernelSU 源码：本仓库 `ksu/drivers_kernelsu/`，同步到 **KernelSU-Next v3.4.2-legacy**
  （含 v3.4.0 审计修复），并保留本机定制（见下）。

为了不破坏 `/vendor/lib/modules`，构建时把 UTS 版本串对齐 stock：

```
4.19.325-cip136-st20-perf-gf914d0b24613
#1 SMP PREEMPT Thu Nov 12 09:32:12 CET 2026
```

> 20261112 ROM 的 stock 配置已用 `scripts/extract-ikconfig` 从其 `boot.img` 提取，
> 与 `config/device.config` 完全一致，构建直接以它为准。工具链按维护者给的
> `build.sh` 换成 AOSP prebuilt `clang-r547379`（clang 20.0.0），
> LTO/CFI/SHADOW_CALL_STACK 与 stock 保持一致（不再关闭）。
>
> `build.sh` 里的 defconfig 链（`vendor/kona-perf_defconfig` + 两个 xiaomi
> fragment）不能照抄：`vendor/xiaomi/alioth.config` 打开了 `CONFIG_CRYPTO_LZ4HC=y`，
> 而 shipped 内核该符号是关的，且这个符号在 4.19 这棵树上根本编不过
> （`crypto/lz4hc.c` 用的是上游名 `LZ4HC_DEFAULT_CLEVEL`，
> `lib/lz4/lz4hc.h` 只定义了 `LZ4HC_CLEVEL_DEFAULT`）。

## 目录

```
.github/workflows/build.yml              # 编译 + 重打包
patches/ksu-newfstat-hook.patch          # fstat(2) 报告追加的 KSU init.rc 长度
patches/cap-checkpoint-restore.patch     # 旧底座用；stock 源码已含，workflow 不再应用
patches/memcg-lru-size-warn.patch        # memcg lru_size WARN 改为 pr_warn，避免 taint
patches/ksu-avc-spoof-hook.patch         # slow_avc_audit 调用 KSU，avc_spoof 才生效
patches/kernelsu-susfs.diff              # KSU-Next/SUSFS 手动 hook 总集成（含 fs/susfs.c）
config/device.config                     # 从设备 /proc/config.gz 导出的完整配置（对照用）
config/ksu.fragment                      # KSU/SUSFS/vermagic 覆盖
ksu/drivers_kernelsu/                    # KernelSU-Next v3.4.2-legacy 源码
```

## 当前补丁

1. **`ksu-newfstat-hook.patch`**
   Android 17 libbase 按 `fstat(2)` 的 `st_size` 精确读取，不再读到 EOF。KSU 的 init.rc
   是追加在 EOF 的，所以给 `newfstat` 加上 `ksu_rc_len + module_rc_len`。

2. **`cap-checkpoint-restore.patch`**（当前不再应用）
   stock 20261112 源码已自带 `CAP_CHECKPOINT_RESTORE(40)` 和 classmap
   `checkpoint_restore`，无需再打；保留文件供旧底座参考。

3. **`memcg-lru-size-warn.patch`**
   4.19 的 `mem_cgroup_update_lru_size()` 在内存压力下会触发 `WARN_ONCE`，给内核打上
   `TAINT_WARN`（`/proc/sys/kernel/tainted` 非 0）。改为 `pr_warn_once`，保留日志和
   `lru_size=0` 重置，但不再 taint。

4. **`ksu-avc-spoof-hook.patch`**
   手工 hook 构建下没有任何地方调用 `ksu_handle_slow_avc_audit()`，KSU 的 `avc_spoof`
   设置成 1 也不会生效。在 `slow_avc_audit()` 入口把 `tsid` 交给 KSU，dmesg 可见
   `avc_spoof/init: slow_avc_audit spoofing enabled!`。

5. **KSU 源码改动**（`ksu/drivers_kernelsu/`）
   - 版本覆盖 `33355` / `v3.4.2`；UAPI 报告 5，匹配 v3.4.2 manager/ksud
   - init.rc 改回 append（排在系统 post-fs-data 之后，等 `installkey /data` 完成）
   - sucompat / stat / faccessat 跳过 `_process_zygote` / `zygote-child`，避免
     `app_zygote -> ksu` 的 bounded transition 失败
   - SUSFS 接入：`susfs_init()`、`reboot` magic 命令分发、policy 加载后解析
     init/zygote/priv_app/ksu SID、post-fs-data 后启动 sdcard 监控
   - SUSFS Kconfig 菜单与 `try_umount` 导出（底座 SUSFS 需要）

## 构建

Workflow 会：

1. 下载 `kernel_src_url` 指定的源码 tarball 到 `kernel/`（默认即上面的 release 资产）
2. 把 `ksu/drivers_kernelsu` 复制到 `kernel/drivers/kernelsu`
3. 应用上面四个 kernel patch
4. 用 stock PixelOS 内核的 UTS 版本字符串构建：
   `#1 SMP PREEMPT Thu Nov 12 09:32:12 CET 2026`，`ci@PixelOS-CI`，
   `KBUILD_BUILD_VERSION=1`
5. 拉取 stock `boot.img`，把 `Image` 重打包成 `boot-ksu.img`
6. 下载 KernelSU-Next v3.4.2 manager（`KernelSU_Next_v3.4.2_33355-release.apk`）

## 使用

仓库：<https://github.com/ziqianou/pixelos-alioth-ksu>

打开 **Actions → Build PixelOS alioth KernelSU kernel → Run workflow**
- `rom_url`：默认 `PixelOS_alioth-17.0-20261112-0826.zip`（download.pixelos.net）
- `kernel_src_url`：默认 `kernel-base-v1` 的源码 tarball
- `jobs`：默认 4

跑完后下载 artifact：`boot-ksu.img`、`Image`、`System.map`、
`KernelSU_Next_v3.4.2_33355-release.apk`。

最近一次已验证构建：run `44543194489`（20261112），产物在 `../artifacts/20261112/`。

> 更新 manager 后先打开一次 app（或确认 `/data/adb/ksud` 已是新版本）再重启，
> 否则该次开机的 ksud 仍是旧 ABI，post-fs-data/模块脚本会被跳过一次。

## 提交与签名

- 提交使用 GPG 签名（EDDSA，key `2B3C9CCF3BB93C93`，`commit.gpgsign=true`）。
- 当前 HEAD：`e83de4a`；`5dd59b4`–`e83de4a` 为 2026-11-19 重签后的提交。
  更早的 `3a58ff6` 是历史无签名提交，保留不动。
- 重签方式：`git rebase --exec 'git commit --amend --no-edit -S' <base>`，
  推送用 `git push --force-with-lease`。
- 非交互环境签名超时时，先让 gpg-agent 缓存口令（如 `echo test | gpg --clearsign`），
  不要用 `-c commit.gpgsign=false` 产生无签名提交。

## 刷入 / 回滚（当前槽 `_b`，PixelOS 20261112）

本机整合后的路径（仓库在 `repo/`，产物在 `../artifacts/`）：

```sh
# 当前已刷：PixelOS 20261112 + KSU 3.4.2 + SUSFS 接线（slot b）
fastboot --set-active=b
fastboot flash boot_b ../artifacts/20261112/boot-ksu.img
fastboot reboot

# 回滚到 20261024（slot a）
fastboot --set-active=a
fastboot flash boot_a ../artifacts/20261118b/boot-ksu.img
fastboot reboot

# 更早的 KSU 3.4.2 / 3.4.0
fastboot flash boot_a ../artifacts/20261118/boot-ksu.img
fastboot flash boot_a ../artifacts/20261024/boot-ksu.img

# 旧 ROM (20260922) 的 KSU boot（slot b 历史备份；已被 20261112 覆盖）
fastboot flash boot_b ../artifacts/uname/pixelos-alioth-ksu-14/boot-ksu.img

# stock/Magisk boot_b 备份（20260922 时代）
fastboot flash boot_b ../artifacts/main/boot_b.backup.img
```

ROM 更新方式（本次 20261024 → 20261112 的 a→b）：

```sh
adb reboot sideload
adb sideload ../downloads/alioth/20261112/PixelOS_alioth-17.0-20261112-0826.zip
# 完成后进 bootloader：切到 b、刷 boot_b、重启
```

> USB 传输不稳时，也可以在已 root 的系统里用 `update_engine_client` 把同一份
> `payload.bin` 写到 inactive slot，再 fastboot 刷 boot。

首次开机后安装 `KernelSU_Next_v3.4.2_33355-release.apk`，授予 root，再按需安装
Zygisk Next、LSPosed、SUSFS 模块等。

模块文件挂载（priv-app 之类）依赖 **metamodule**：KernelSU-Next v3.4.x 自身不再做
Magic Mount，需要在管理器里安装 `meta-overlayfs` 之类的 metamodule。本机用
`meta-overlayfs v1.3.1`，OpenEUICC 的 `system_ext` 内容在其 ext4 镜像里。

## 本地工作区

本仓库所在的工作区还包含（不随 git 推送）：

```
../artifacts/          # 各次构建产物；20261112=当前(20261112 ROM), 20261118b=20261024 SUSFS 接线,
                       # 20261118=驱动同步, 20261024=KSU 3.4.0, uname=旧 ROM, main=回滚备份
../cache/alioth-work/  # 本地内核源码/clang/工具/ROM dump
../downloads/          # ROM、boot、vbmeta、LSPosed、KernelSU APK 等
```

详见 `../README.md`。

## 设备侧状态 / 已知限制

- 当前 ROM：PixelOS 20261112，**slot b**；system 构建时间 Thu Nov 12 09:22:50 CET 2026，
  incremental 1794471770，安全补丁 2026-11-01
  （运行时 `ro.build.*` 可能被认证属性模块伪装，真实值看 `ro.system.build.*` 或 `/system/build.prop`）
- 当前 boot：`../artifacts/20261112/boot-ksu.img`（SHA256 `478123ea…de1e9f32`）
- 内核：`4.19.325-cip136-st20-perf-gf914d0b24613`，`uname -v` 与 stock 一致
- KSU built-in v3.4.2（33355），ksud/manager v3.4.2（33355，uapi 5）；Zygisk Next 1.5.2；LSPosed v2.2.1-log 7944
- metamodule：`meta-overlayfs v1.3.1`；OpenEUICC 以 priv-app 安装，`SYSTEM|PRIVILEGED`，四个特权权限 granted
- SUSFS：内核 v2.3.0，命令入口/初始化/SID/sdcard 监控均已接通；`set_uname`、`add_sus_path`、
  `hide_sus_mnts_for_non_su_procs`、`enable_log`、`show version/features` 实测可用
- `cap_last_cap=40`；`tainted=4`（仅 `S`）
- avc_spoof 已真实生效（之前缺 `slow_avc_audit` hook，等于没开）
- LSPosed 首次刷内核后开机可能遇到 system_server 未连接（`lspctl status` 显示
  `System server: not connected`，日志 `failed to prepare for SystemServer`），
  是启动竞态；再重启一次即可，之后模块正常注入
- 音频：grill 快照上 A2DP offload 会让 ADSP 固件 fatal（蓝牙无声）；已切到 stock 源码，待复测
- `reqable-magisk`、`TA_enhanced` 当前 **disabled**；`Automatic_brick_rescue` 是救砖模块
- 厂商模块 `rmnet_perf` / `rmnet_shs` / `gspca_main` 仍无法加载：
  `disagrees about version of symbol module_layout`（`CONFIG_MODVERSIONS` 的 CRC 与
  私有 stock 内核不一致）。三者都不是开机必需，本次开机也没有触发加载
- `untrusted_app` 读取 `/proc/version`、`/proc/sys/kernel/osrelease` 的 SELinux 规则
  在设备的 `/data/adb/modules/susfs4ksu/sepolicy.rule`（不在仓库）
- SUSFS 底座 Kconfig 没有新版 `AUTO_ADD_*` 选项，WebUI 中对应开关不会显示；
  旧命令（`add_sus_mount`、`sus_su` 等）被忽略
- Hunter（旧 ROM）：87/97 通过；Kernel Runtime 剩余原因是 `rawMode=libc_syscall_fallback`；
  新 ROM 未完整复测
- Play Integrity：BASIC 通过，DEVICE/STRONG 受 keybox/TEE 限制，未解决

## OTA

内置 KSU 内核会被 PixelOS OTA 覆盖。更新后：

- 用新 ROM 的 zip URL 重新跑一次 workflow；
- OTA 装到 inactive slot 后先别重启，把新的 `boot-ksu.img` 刷到**重启后将使用的 slot**；
  或等 OTA 完成后再进 fastboot 刷当前 slot（本次为 `boot_b`，注意 slot 可能变化）。

> 内置 KSU 无法使用 KernelSU 管理器的 "install to inactive slot" 功能。

## 已知风险

- 内核源码现为 PixelOS-Devices 的 stock 提交 `f914d0b`，vermagic 与 ROM 一致，但
  `/vendor/lib/modules` 的 `rmnet_perf`/`rmnet_shs`/`gspca_main` **加载失败**：
  `disagrees about version of symbol module_layout`（`struct module` 布局与出包内核
  不同，说明 `f914d0b` 与出包源码仍有差异，对应 `techpack/audio` 那 3 个文件）。
  实测不影响开机与数据网络，只影响网络性能模块和 UVC 摄像头驱动。
- 源码现直接使用 ROM 对应的 stock 提交 `f914d0b`；后续 OTA 需同步换成新 ROM 的
  stock 提交（GitHub 上按 `PixelOS-Devices` 的提交取 archive）。
- LTO/CFI/SHADOW_CALL_STACK 与 stock 一致，实测可正常开机：`/proc/config.gz` 里三项
  均为 `=y`，dmesg 可见 `cfi_jt` 符号。构建成本也比预期低——`Build Image` 约 14 分钟
  （ccache 命中约 45%），整个 workflow 约 24 分钟；换环境若 OOM，回退方案是
  `CONFIG_LTO_NONE=y`。
- 已在真机 alioth / slot b（PixelOS 20261112）验证：LTO+CFI 内核开机约 28 秒、
  KSU 3.4.2 root、13 个模块加载、Zygisk Next 注入 LSPosed、SUSFS 命令；
  slot a 保留 20261024 作回滚。
- 若 PixelOS 后续把内核升到 `cip137/st21`，需要重新生成源码快照并核对 vermagic。

## 刷机与救砖备忘（2026-11-22 实测）

### 事故链速查

`Automatic_brick_rescue` 被触发 → `chmod 000` 掉 `post-fs-data.d`/`service.d`（Zygisk 失效）
\+ 给所有模块 `touch disable` → 启动计数 ≥4 后每次开机无条件 `reboot` → 连续启动失败
把 `slot-retry-count:b` 耗尽、`b` 变 `unbootable` → 自动切到 **旧系统的 `a` 槽** →
旧系统配不上新 `/data`，继续无限重启。**内核本身始终是好的。**

### `fastboot boot` 在这台机器上是空操作

`fastboot boot <img>` 会返回 `Booting OKAY`，但**不会加载传入的镜像**，设备只是重启进
分区里的内核。验证方法：启动后看 `/proc/version`，显示的仍是分区内核的编译器 banner。

→ 任何「内存引导验证内核」的结论都不可信，只能靠 `fastboot flash boot_<slot>` 验证。

### 槽位降级会让所有启动都失败

每次启动失败，bootloader 会把该槽的 `slot-retry-count` 减一；归零后标
`slot-unbootable=yes`、自动切到另一个槽。若另一个槽是**旧版本系统**，它配不上已被
新 ROM 升级过的 `/data`，表现就是「第一屏 → 约 20 秒 → 黑屏重启」的无限循环，
而且此时无论刷什么内核都同样失败（因为系统是旧的那个）。

恢复：

```sh
fastboot --set-active=b          # 复位槽位 + retry 计数
fastboot flash boot_b <boot.img> # 刷回已知可启动的内核
fastboot reboot
```

### `Automatic_brick_rescue` 模块会放大故障

该 Magisk/KSU 模块（搞机助手生成）的逻辑：

- `post-fs-data`：启动计数 `Number_of_starts.log`，等于 2 时 `chmod 000
  service.d/*` 并**禁用所有模块 + reboot**；≥4 时每次开机无条件 `reboot`；
- `service.sh`：开机 3 分钟后若 `init.svc.bootanim` 仍不是 `stopped`，**禁用所有模块 +
  reboot**。

所以只要有一次启动没在 3 分钟内完成开机动画，它就会把所有模块打上 `disable` 并主动
重启，计数到 4 之后变成无限重启。它还会把 `post-fs-data.d` / `service.d` 里的启动脚本
`chmod 000`，从而顺带废掉 Zygisk（见下一节）。

排查内核问题时建议先禁用该模块并复位计数，事后**直接卸载**它：

```sh
su -c 'touch /data/adb/modules/Automatic_brick_rescue/disable'   # 临时禁用
su -c 'echo 0 > /data/adb/modules/Automatic_brick_rescue/Number_of_starts.log'
```

### Zygisk Next 注入失败：先查脚本权限，不要怀疑内核

现象：`/data/adb/zygisksu/.abort_msg` 出现 `❌ Last injection failed!` 或
`❌ Module files corrupted`，模块自己写 `disable` 自禁用，系统里没有 `zn-*` 进程，
LSPosed 失效；`znctl status` 报 `failed to connect to server`（说明 zygiskd 没起来）。

真因（本次实测）：救砖模块执行过的
`chmod 000 /data/adb/post-fs-data.d/* /data/adb/service.d/*` 把
`00-zygisk.sh`（内容为 `setsid ./bin/zygiskd daemon`）也设成了不可读 → 守护进程从未
启动 → 注入必然失败。同理 `hide_leaks.sh`、`pi_props.sh`、`adb-guard.sh`、
`battery_unlock.sh` 也一起失效，只是没人注意。

**与 LTO/CFI 无关**：修好权限后，同一套 LTO+CFI 内核（clang 20.0.0）上
Zygisk Next 1.5.3 正常注入，`module.prop` 显示
`[Root: ✅KernelSU (33355), UM, ZL, 1 service(s) injected]`，进程里有
`zn-nsdaemon-zygote`、`zn-nsdaemon-zygote_secondary`、`zn-zygisk-companion64`。

修复：

```sh
su -c 'chmod 755 /data/adb/post-fs-data.d/*.sh /data/adb/service.d/*.sh'
su -c 'mv /data/adb/modules/zygisksu/disable /data/local/tmp/zygisksu.disable.off'
su -c 'mkdir -p /data/local/tmp/mod-backup && \
       mv /data/adb/modules/*/disable.off /data/local/tmp/mod-backup/ 2>/dev/null'  # 清残留
reboot
```

已知残留问题：Zygisk Next 仍会**偶发**在开机时自禁用，同一内核下一次开机又正常，报错
在 `Last injection failed!` / `Module files corrupted` 之间跳变。用官方
`Zygisk-Next-1.5.3-880-*.zip` 走 `ksud module install` 干净重装也无效，怀疑与
`meta-overlayfs`（metamodule）/ `susfs4ksu` 的挂载时序或可见性有关；需要稳定复现时先
把这两个模块关掉再测。日常恢复：KSU 管理器里把 Zygisk Next 关一次再开，或挪走
`disable` 后重启。

## 反 root 检测（Hunter 6.72）与内核侧修复

实测设备用 `com.zhenxi.hunter` 扫描（28 项，高风险 2 项）。三项可操作的问题及处理：

### Kernel Tainted（`/proc/sys/kernel/tainted = 4`）

来源是 `arch/arm64/kernel/cpufeature.c` 的
`add_taint(TAINT_CPU_OUT_OF_SPEC, ...)`（taint 位 2），dmesg 表现为：

```
[    0.014723] CPU features: Unsupported CPU feature variation detected.
```

Qualcomm 的 big.LITTLE 特性表本来就对不上，所以每次开机必打这个标记。taint **不能从
用户态清除**（`echo 0 > /proc/sys/kernel/tainted` 被内核忽略，该 sysctl 只置位不清位），
只能改内核：`patches/no-cpu-taint.patch` 保留 warning、去掉 `add_taint()`，刷入后
`tainted = 0`。

### Find Root Mark In Mountinfo（`OVERLAYFS_SUPER_MAGIC`）

KSU 会把模块的 overlay 挂载从普通进程的 `mountinfo` 里隐藏，但 `statfs()` 仍返回
overlay magic，检测器一比就暴露：

```console
# shell(uid 2000)
grep -ci overlay /proc/self/mountinfo   -> 0
stat -f /system                         -> Type: overlay     # 不一致
```

修复是启用 `CONFIG_KSU_SUSFS_SUS_OVERLAYFS`（让非 root 进程看到底层 fs）：

```console
stat -f /system   -> Type: ext3/4        # root 和 shell 一致
```

两个坑：

1. 这个符号**没有定义在 `ksu/drivers_kernelsu/Kconfig` 里**，只在 `fs/overlayfs/*`
   的 `#ifdef` 中引用。不补 Kconfig 条目的话，`CONFIG_...=y` 会被 `olddefconfig`
   静默丢弃（构建不报错，但功能没生效）。
2. 同一套 patch 里的 `CONFIG_KSU_SUSFS_AUTO_ADD_SUS_KSU_DEFAULT_MOUNT` /
   `..._AUTO_ADD_SUS_BIND_MOUNT` **只有声明和调用、没有实现**，打开会链接失败，别开。

### SELinux Policy Oracle Untrusted ×2

`reasonCodes=[CONTEXT_CHANNEL_DISAGREEMENT:3,4,5,10,11]`：同一 SELinux 上下文经不同
通道读出的结果不一致。这是 KSU 改写内核 sepolicy 的必然结果，本 patch 版本的 SUSFS 也
没有 `AVC_LOG_SPOOFING`（`ksu_susfs --help` 里没有该命令，`show enabled_features`
里也没有），要消除只能换更新版 SUSFS 底座，收益不大。

### 顺带：LTO 链接期的 lz4 栈帧警告

```console
ld.lld: warning: ../lib/lz4/lz4.c:1370:0: stack frame size (16496) exceeds limit (4096)
```

`CONFIG_FRAME_WARN=4096` 在 LTO 下由 `ld.lld` 在链接期报出，链接期警告不吃 `-Werror`，
不影响产物；官方内核（同 LTO+CFI+同配置）也会报同样的两条，无需处理。

### 相关设备侧修复（非内核）

- `pi_props.sh` 只找 Magisk 的 `resetprop`，在 KSU 上直接 `exit 0`，导致
  PIF 属性（fingerprint / 安全补丁 / verifiedbootstate）从未生效；已加
  `/data/adb/ksu/bin/resetprop` 回退。
- Tricky Store 守护进程启动失败（`verify1 failed` / `unverified!`）时，用同版本官方
  包 `ksud module install` 重装即可恢复。

### cmdline 伪装会引入新矛盾（已回滚）

SUSFS 的 `spoof_cmdline=1` 只改 `/proc/cmdline`，改不到
`/proc/device-tree/chosen/bootargs`：

```console
/proc/cmdline                     -> androidboot.verifiedbootstate=green   (伪装)
/proc/device-tree/chosen/bootargs -> androidboot.verifiedbootstate=orange  (真实)
```

两者不一致会被 Hunter 的 `Kernel Runtime Consistency Incomplete` 抓到（高风险项从
9 变 10），所以保持 `spoof_cmdline=0`。

`ro.boot.verifiedbootstate=green` 仍然必须用 `resetprop` 设置（Play Integrity 需要
locked 状态），代价是 `Find Prop Modify Mark` 里这一项无法消除；`ro.build.tags` /
`ro.build.type` 这类本来就正确的属性不要再写（每写一次都会 bump prop serial）。

### 不要在内核里改 androidboot.verifiedbootstate（实测会挂）

曾尝试在 `arch/arm64/kernel/setup.c` 里把 `boot_command_line` 和
`/chosen/bootargs` 的 `verifiedbootstate=orange` 就地改成 `green`（连同
`flash.locked` / `vbmeta.device_state`），让 init 原生设置 `ro.boot.*`。结果**卡在第一屏**，
无法进入系统。

原因：这些 `androidboot.*` 参数不只是给用户态看的，init 会据此决定是否按 locked/AVB
校验流程挂载分区。太早把它们改成 locked/green，会让 init 走签名校验路径并失败。
所以 `ro.boot.verifiedbootstate=green` 只能在 init 之后用 `resetprop` 改（代价是那项
prop serial 检测无法消除），不要在启动早期碰它。
