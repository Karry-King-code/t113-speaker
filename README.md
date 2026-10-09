# 考核⑤ 喇叭：播放音频

## 状态

| 项目 | 状态 |
|---|---|
| 声卡驱动 | ✅ 已就绪（`audiocodec` 已注册） |
| 播放通路 | ✅ 已验证（DAPM 全 On） |
| HPOUT 使能/音量 | ✅ 已配置 |
| **功放使能 PA_SHDN** | ✅ 已定位（PB2 = gpio-34）并拉高 |
| 开机自动配置 | ✅ 已写入 `/etc/rc.local`，重启验证通过 |
| 测试音频 | ✅ 已生成并放入板子 `/root/audio/` |
| **听到声音** | ⏳ **待用户插喇叭确认** |

---

## ★ 完整复刻教程

### 👉 [`doc/喇叭-播放音频全程教程.md`](doc/喇叭-播放音频全程教程.md)

含完整诊断链路（从"播放没报错但没声音"追到"功放使能脚被下拉"）、
原理图解读、时序陷阱、开机脚本、测试音频生成、老师追问预演、自检清单。

---

## 核心结论（一句话）

> **喇叭不出声，不是驱动问题，是功放 LM4871 的使能脚 `PA_SHDN`（接 PB2）
> 被 100kΩ 电阻默认下拉关闭了，必须软件主动拉高。**

---

## 硬件事实

| 项目 | 值 | 来源 |
|---|---|---|
| 音频芯片 | `audiocodec`（T113 内置） | `/proc/asound/cards` |
| 功放 | **LM4871**（3W 音频功放） | 原理图第 5 页 |
| 功放使能脚 | **PA_SHDN** | 原理图第 5 页 |
| 使能脚引脚 | **PB2** | 原理图第 1 页（引脚 86） |
| 全局 GPIO 号 | **34**（1×32+2） | 计算 + 已知项交叉验证 |
| 喇叭座 | **CN2**（HC-1.25-2PWT） | 原理图第 5 页 |
| 信号来源 | HPOUTR（经 R136 20kΩ） | 原理图第 5 页 |
| 默认状态 | **关闭**（R138 100kΩ 下拉） | 原理图第 5 页 |

---

## 速查：三条核心命令

```sh
# ① 拉高功放使能（必须！否则喇叭不响）
adb shell "echo 34 > /sys/class/gpio/export; echo out > /sys/class/gpio/gpio34/direction; echo 1 > /sys/class/gpio/gpio34/value"

# ② 设置音量
adb shell "amixer cset name='HPOUT Switch' 1; amixer cset name='HPOUT Gain' 7; amixer cset name='DACL Volume' 200; amixer cset name='DACR Volume' 200"

# ③ 播放
adb shell "aplay /root/audio/test_440.wav"
```

> Windows 侧 adb：`D:\t113_work\flash_tools\adb.exe`
> Git Bash 里要加 `MSYS_NO_PATHCONV=1` 前缀

---

## 目录说明

| 文件 | 内容 |
|---|---|
| `doc/喇叭-播放音频全程教程.md` | ★ **完整复刻教程** |
| `patches/rc_local_new.sh` | 开机自动配置脚本（音频 + 显示） |
| `patches/gen_test_audio.py` | 生成测试音频的 Python 脚本 |
| `patches/test_440.wav` | 440Hz 标准音 A 测试文件 |
| `patches/test_scale.wav` | 音阶测试文件（辨识度更高） |

---

## 你明天要做的（3 步）

### 1. 插喇叭
把喇叭插到板子的 **CN2** 座（2P 1.25mm）。

### 2. 烧录镜像（或直接用现板）
**好消息**：`rc.local` 的改动**不需要重新编译烧录** ——
它已经写进板子的 `/overlay` 持久层，**重启就生效**。

如果你重新烧录了固件，需要重新装一次 `rc.local`（步骤见教程第 5 部分）。

### 3. 播放测试

```sh
adb shell "aplay /root/audio/test_scale.wav"
```

**听到音阶声 = 考核⑤完成。**

---

## 如果没声音，按这个顺序排查

| 顺序 | 检查 | 命令 | 期望 |
|---|---|---|---|
| 1 | 功放使能 | `cat /sys/class/gpio/gpio34/value` | `1` |
| 2 | HPOUT 开关 | `amixer cget name='HPOUT Switch'` | `values=on` |
| 3 | 音量 | `amixer cget name='HPOUT Gain'` | `values=7` |
| 4 | 播放通路 | 播放中看 DAPM | `DACL/DACR/HPOUT` = `On` |
| 5 | 音频文件 | `ls /root/audio/` | 文件存在 |
| 6 | **物理连接** | 看喇叭是否插在 CN2 | 插紧 |

> **第 1 项是最常见的漏项**（重启后 gpio-34 需要约 28 秒才被 rc.local 拉高）。

---

## 下一步

考核⑥ MIC（录音回放）—— 好处：**共用同一张声卡**，
`rc.local` 里已经配好 `MIC3 Input Select/Switch/ADC3 Gain` 了。

---

*最后更新：2026-10-09 ｜ 软件侧全部验证通过，待听声*
