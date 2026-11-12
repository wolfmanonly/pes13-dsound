# pes13-dsound
PES 2013 Switch 移植版（FEXTendo / Wine-NX）**短音效丢失**问题的实验性 `dsound.dll` 修复与诊断记录。
> 非官方项目，与 FEXTendo、Konami、CRI ADX 均无关联。  
> 当前仍是**部分修复**，不是最终方案；请先读完“当前状态”再使用。
**语言 / Language:** [中文](README.md) | [English](README.en.md)
---
## 当前状态（2026-11-12）
| 声音 | 原版 baseline | 应用本仓库补丁后 |
|---|---|---|
| BGM / 解说 / 观众 | 正常 | 正常 |
| 哨声 / 踢球 / 触球 | **无声** | 部分恢复，仍不稳定 |
| 菜单切换声 | **无声** | 前两次按键有声，第三次开始无声 |
结论：
- 原始问题是**所有短 6ch 音效**（哨声、踢球、菜单 UI）都听不到，不是只有踢球。
- 补丁能把其中一部分恢复到可听，但 **ADX 的 voice 复用/循环行为**仍会导致第三个及之后的短音效丢失。
- 这不是音量滑块、`settings.dat` 或原生 Windows `dsound.dll` 能解决的问题。
---
## 问题现象
- 同一份移植包在 PC（Windows 原生 DirectSound）上声音完整。
- Switch 上：
  - BGM、解说、观众助威正常；
  - 哨声、踢球、触球、菜单切换声等**短音效**无声；
  - 游戏内“比赛音效音量”拉到最大无效；
  - 换原生 Windows `dsound.dll` 会卡死，无法进游戏。
- 用调试版 `dsound.dll` 抓到的 6ch buffer **有真实 PCM 数据**，但音量被设得很低：
```
fmt tag=0xFFFE ch=6 rate=48000 bits=16
buflen=49152  (约 85ms)
flags=0x000580A0
vol=-2570  或  vol=-10000
```
`0x000580A0` 解码：
```
DSBCAPS_GLOBALFOCUS | DSBCAPS_GETCURRENTPOSITION2 |
DSBCAPS_LOCDEFER | DSBCAPS_CTRLVOLUME | DSBCAPS_CTRLFREQUENCY
```
没有 `DSBCAPS_CTRL3D`，也没有 `DSBCAPS_LOCHARDWARE`。  
也就是说：不是 3D 定位、不是硬件 voice 申请失败，而是**普通 6 声道短 buffer + 极低音量/循环状态**。
---
## 测试环境
- Switch（ Atmosphere / 自制系统）
- FEXTendo `0.4.2`
- Wine-NX `5.3.1.904`（`autorunhq/autorun` @ `1bc4e45163f0d2328cdfd35c7f471dd9821bb879`）
- FEX-Emu（x86 → ARM64）
- Box64 `dae0917c47b4edd8956f314210417a20fd225c4b`
- 游戏：PES 2013 中文版
- 原始 `dsound.dll`：
  - 大小 `1,200,128` 字节
  - SHA256 `6258E908BC589AD916268655B4892840900DD7584A4FDDF8AE36F9E91FFE46A6`
---
## 仓库结构
```
.github/workflows/build-dsound.yml   GitHub Actions 构建
patches/
  dsound-fix.patch                   primary flags + LOCHARDWARE 回退
  dsound-lfe.patch                   5.1/7.1 LFE 折进左右
  dsound-stereo.patch                强制设备为 2 声道
  dsound-dump.patch                  诊断 + 短 6ch 音量/单次播放（合并版）
  dsound-probe.patch                 logging 版：写 dsound-probe.log
  dsound-hwcaps.patch                非 logging 版：只改 caps
```
---
## 构建
1. Fork / 上传本仓库到 GitHub。
2. Actions → `build-dsound` → **Run workflow**。
3. 参数：
   - `logging: true`：带 `C:\dsound-probe.log` 日志，适合诊断。
   - `logging: false`：只应用 caps 修改，日志更少。
4. 完成后下载 artifact：
   - `dsound-dll`
   - `dsound-source`
构建只编译 `dsound.dll`，不会重新编译 Mesa/DXVK，通常 2 分钟内完成。
---
## 补丁说明
### `dsound-fix.patch`
- 复用 primary buffer 时合并能力位（`CTRL3D` / `CTRLVOLUME` 等）。
- 非 primary 的 `DSBCAPS_LOCHARDWARE` 不再直接返回 `DSERR_UNSUPPORTED`，改为软件缓冲。
> 早期怀疑方向；日志显示游戏实际没有请求 `LOCHARDWARE` / `CTRL3D`，因此这两个改动不是根因，但保留无副作用。
### `dsound-lfe.patch`
把 `put_surround512stereo` / `put_surround712stereo` 里原本被丢弃的 LFE 折进左右声道：
```c
case 3:
    value *= 0.7f;
    dsb->put_aux(dsb, pos, 0, value);
    dsb->put_aux(dsb, pos, 1, value);
    break;
```
### `dsound-stereo.patch`
在 `DSOUND_WaveFormat()` 末尾把设备格式压到 2 声道：
```c
if (w->nChannels > 2) {
    w->nChannels = 2;
    w->nBlockAlign = ...;
    w->nAvgBytesPerSec = ...;
    w->dwChannelMask = SPEAKER_FRONT_LEFT | SPEAKER_FRONT_RIGHT;
}
```
这样 6ch/8ch secondary buffer 会走 Wine 的下混路径，而不是把 5.1 数据直接送给只有 2 声道的 audout。
### `dsound-dump.patch`（当前合并版）
包含四部分：
1. **逐 voice 快照**
   - 每个 6ch buffer 第一次有明显数据时写：
     ```
     drive_c\dsound-voice-01.raw
     drive_c\dsound-voice-01.txt
     ```
   - txt 内容：
     ```
     seq peak vol pan flags ch0..ch5
     ```
2. **2 秒最终混音录音**
   - 6ch 写入峰值 > 3000 时触发；
   - 连续录 `DSOUND_PerformMix` 的最终输出 2 秒：
     ```
     drive_c\dsound-mix.raw
     drive_c\dsound-mix.txt
     ```
   - txt 格式示例：
     ```
     rate=48000 ch=2 bits=32 tag=0xfffe block=8
     ```
3. **短 6ch 单次播放**
   - 在 `DSOUND_MixInBuffer()` 里清掉 `DSBPLAY_LOOPING`：
     ```c
     if (dsb->pwfx->nChannels == 6 && dsb->buflen <= 65536)
         dsb->playflags &= ~DSBPLAY_LOOPING;
     ```
   - 目的：让 85ms 的短音效播完就结束，释放 voice。
4. **定向音量强制**
   - 只对“短 6ch、音量在 -15 ~ -90 dB”的音效在混音时强制 `1.0`：
     ```c
     nChannels == 6 && buflen <= 65536 &&
     vol < -1500 && vol > -9000
     ```
   - BGM 静音 `-10000` 不强制，正常音乐 `-1070` 不强制。
> 这是诊断 + 实验性修复的混合补丁，会写文件、会改音量；不适合作为最终生产补丁。
### `dsound-probe.patch`
- 把 `DSCAPS` 的 3D/硬件 voice 数量报成 32。
- 记录 `caps` / `create` / `Play` / `QueryInterface` 到：
  ```
  C:\dsound-probe.log
  ```
- 单次运行最多 4000 行，但文件是追加模式，多次启动会累积；测试前建议删除。
### `dsound-hwcaps.patch`
只改 caps，不写日志；用于不希望产生 probe 日志的场景。
---
## 诊断发现
### 1. 不是 3D，也不是硬件 voice
probe 日志里所有 buffer：
```
flags=0x000580A0
ctrl3d=0
```
没有 `CTRL3D`，也没有 `LOCHARDWARE`。  
创建、`QueryInterface(IDirectSoundBuffer8)`、`Play` 都成功。
### 2. 音效是 6 声道短 buffer
```
tag=0xFFFE (WAVE_FORMAT_EXTENSIBLE)
ch=6, rate=48000, bits=16
buflen=49152 → 4096 帧 × 6ch × 2B ≈ 85ms
```
还有一条 8ch / 102400 字节的 buffer，可能是 BGM/环境流。
### 3. 数据存在，但音量极低
抓到的 raw PCM 有真实波形，例如：
```
ch0 rms=8788 max=31000
ch1 rms=8550 max=31052
ch2..5 = 0
```
但对应 `vol` 常见为：
```
-2570   约 -25.7 dB
-10000  约 -100 dB，等于静音
```
`-10000` 是 `DSBVOLUME_MIN`；这也是原始版本连菜单前两声都听不到的原因。
### 4. 强制音量后，前两次菜单声恢复，第三次仍丢失
在混音器里强制 6ch 短音效为 `1.0` 后：
- 主菜单前两次按键有声；
- 第三次开始没有；
- `dsound-probe.log` 显示 `Play` 被调用几百次，但几乎都集中在两个 buffer 指针：
  ```
  play buf=1067AE80 flags=0x00000001 ctrl3d=0
  play buf=0AE95BB8 flags=0x00000001 ctrl3d=0
  ```
### 5. 当前推测
可能是以下两种情况之一：
1. **ADX 自己的 UI voice 池只有 2 个**，第三个事件被中间件丢弃；
2. **Wine 的 buffer 状态/循环语义**让 ADX 认为 voice 一直忙，反复 `Play(LOOPING)`，第三个事件拿不到 voice。
这不是确定结论，需要更深入的 ADX 行为分析。
---
## 上机测试
1. 备份 Switch 上的：
   ```
   drive_c\windows\syswow64\dsound.dll
   ```
2. 用 artifact 里的 `dsound.dll` 覆盖。
3. 启动器有运行时校验：
   - 先让原版启动一次，生成 `launcher\startup-verified.dat`；
   - 再替换，缓存有效时不会触发全量 SHA 校验。
4. 测试前删除旧的诊断文件：
   ```
   drive_c\dsound-probe.log
   drive_c\dsound-mix.raw
   drive_c\dsound-mix.txt
   drive_c\dsound-voice-*.raw
   drive_c\dsound-voice-*.txt
   ```
5. 进主菜单连续按键、进训练踢球，观察声音与生成的文件。
---
## 已知限制
- 第三个及之后的短音效仍可能丢失；根因未完全确认。
- `dsound-dump.patch` 会写诊断文件、强制部分音量，不适合长期使用。
- 强制 6ch 音量可能让 BGM/环境声变响；最新版已移除会覆盖 BGM 静音的 `SetVolume` 钳制，但 BGM 控制仍需实测。
- 仅测试于 FEXTendo 0.4.2 + 特定 Wine-NX/Box64 版本，其他版本行为可能不同。
- 不保证不触发启动器校验/自动修复；替换前务必备份。
---
## 如何报 bug
最对口的仓库是 FEXTendo 的 issue：
- <https://github.com/Ibnuard/pes13_nx/issues/12>
建议附上：
- `dsound-probe.log`
- `dsound-mix.raw` / `dsound-mix.txt`
- `dsound-voice-*.raw` / `.txt`
- 本仓库的 `dsound-dump.patch`
- 测试步骤、预期/实际结果
描述时建议写成观察，而不是结论：
> After forcing 6-channel volume to full scale, the first two menu UI sounds are audible, the third and later ones are not. The probe log shows repeated `Play(DSBPLAY_LOOPING)` on only two buffer pointers, suggesting either an ADX-internal voice limit or Wine's buffer status/looping semantics preventing voice reuse.
如果能在**原版 Wine**（非 Wine-NX）复现，也可以报到 Wine Bugzilla 的 `dsound` 组件；否则上游很可能要求先提供 vanilla Wine 复现。
---
## 免责声明
- 本项目仅为个人研究/存档，不提供任何担保。
- 修改系统 DLL 有导致游戏无法启动、存档异常、启动器自动修复等风险。
- 请自行备份 `dsound.dll` 与存档。
- 与 Konami、CRI、FEXTendo、Wine 项目均无隶属关系。
