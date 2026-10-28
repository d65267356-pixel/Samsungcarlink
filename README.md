# Samsung CarLink Adaptation

三星车联（CarLink）投屏音频修复模块与使用说明。

## 文件

| 文件 | 说明 |
|---|---|
| `carlink_audio_hook.apk` | CarLink Audio Hook 模块（`com.example.clinkhook` 1.0，约 20.5 KB） |
| `使用说明.txt` | 原作者使用说明（安装、验证、限制、回滚） |

APK SHA-256：

```
FFB728E4142393B3FDF60B35876929EC0FF897A2B0D64923ACD8F98220D58EA4
```

## 模块作用

三星车联在 ICCOA 模式下会把手机媒体声音吸进 `REMOTE_SUBMIX`（准备通过投屏通道发给车机）。部分车机不渲染这条投屏音频流，表现为「画面正常、车机没声音、手机也没声音」。

模块在车联进程内运行一个守护线程：

1. 检测到 `REMOTE_SUBMIX` 被注册时，把车机蓝牙 A2DP 连回来；
2. 把 Android「媒体」音频策略的优先设备设为该 A2DP，让媒体声音走蓝牙而不是投屏通道。

## 安装（摘要）

1. 安装 `carlink_audio_hook.apk`；
2. LSPosed 管理器 → 模块 → 启用 **CarLink Audio Hook**；
3. 作用域勾选 **三星车联（`com.samsung.android.carlink`）**；
4. 重启手机，或强制停止三星车联使其重新加载模块；
5. 连接车机、播放音乐；首次连接后等待 5～10 秒。

完整步骤、验证方法和已知限制见 [`使用说明.txt`](使用说明.txt)。

## 实测补充（非原作者说明）

在以下环境实测通过（有声音）：

- Samsung SM-S9280（Galaxy S24 Ultra），Android 16 / API 36，港版 TGY；
- 三星车联 3.3.00.10（普通安装应用，非系统特权应用）；
- LSPosed 2.1.1 + Zygisk Next，KernelSU root；
- 车机：智己 LS6（ICCOA 无线车联）。

需要注意：车联作为普通安装应用默认拿不到 `android.permission.MODIFY_AUDIO_ROUTING`（protectionLevel 为 `signature|privileged|role`）。缺少该权限时：

- 车联自己的音频路由调用会抛 `SecurityException: Not allowed to audio routing`，甚至导致车联进程崩溃；
- 模块设置「媒体优先设备」的调用也会被系统拒绝，表现为装了模块仍然没声音。

在本机通过 root 授予后模块立即生效：

```bash
adb shell su -c "pm grant com.samsung.android.carlink android.permission.MODIFY_AUDIO_ROUTING"
```

验证路由是否生效：

```bash
adb shell dumpsys audio | sed -n '/Preferred devices for strategy/,+3p'
adb shell dumpsys media.audio_policy | sed -n '/Device role per product strategy dump/,+3p'
```

应能看到 `type:bt_a2dp addr:<车机 MAC>` 与 `AUDIO_DEVICE_OUT_BLUETOOTH_A2DP`。

> 该授权属于非官方改动，车联升级、卸载重装或系统更新后可能丢失，需要重新授予；是否长期有效请自行评估。

## 卸载 / 回滚

1. LSPosed 管理器中关闭本模块；
2. 卸载 APK；
3. 重启手机。

如需撤销权限：

```bash
adb shell su -c "pm revoke com.samsung.android.carlink android.permission.MODIFY_AUDIO_ROUTING"
```

## 免责声明

本模块通过 LSPosed 在三星车联进程内运行，仅调用标准 Android 音频 API 调整媒体路由，不修改系统分区、不注入车机。仍属非官方改动，使用前请自行评估风险；若车联或 LSPosed 升级后出现异常，请先关闭模块再排查。
