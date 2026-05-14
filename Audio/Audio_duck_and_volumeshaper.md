# Android 音频系统 Duck 机制与 VolumeShaper 深度解析

> **会话日期**: 2026-05-13  
> **项目路径**: `/home/.../android12`  
> **主题**: AudioTrack、VolumeShaper、系统 Duck 机制实现原理

---

## 目录

- [一、AudioTrack 基础](#一audiotrack-基础)
  - [1.1 AudioTrack 类定义](#11-audiotrack-类定义)
  - [1.2 核心功能](#12-核心功能)
- [二、VolumeShaper 原理](#二volumeshaper-原理)
  - [2.1 VolumeShaper 概述](#21-volumeshaper-概述)
  - [2.2 核心参数](#22-核心参数)
  - [2.3 插值类型](#23-插值类型)
- [三、系统 Duck 机制](#三系统-duck-机制)
  - [3.1 什么是 Duck](#31-什么是-duck)
  - [3.2 Duck 的两层实现](#32-duck-的两层实现)
  - [3.3 AudioMatrix 交互矩阵](#33-audiomatrix-交互矩阵)
  - [3.4 Duck 决策流程](#34-duck-决策流程)
- [四、VolumeShaper 触发机制](#四volumeshaper-触发机制)
  - [4.1 核心发现](#41-核心发现)
  - [4.2 Duck 曲线配置](#42-duck-曲线配置)
  - [4.3 完整触发流程](#43-完整触发流程)
  - [4.4 取消 Duck 恢复音量](#44-取消-duck-恢复音量)
- [五、自定义 Duck 曲线和时长](#五自定义-duck-曲线和时长)
  - [5.1 方案一：修改系统默认配置](#51-方案一修改系统默认配置)
  - [5.2 方案二：车载系统定制](#52-方案二车载系统定制)
  - [5.3 方案三：运行时动态调整](#53-方案三运行时动态调整)
  - [5.4 方案四：应用层自定义](#54-方案四应用层自定义)
- [六、company 多 AudioBus 硬件路由架构](#六company-多-audiobus-硬件路由架构)
  - [6.1 传统手机 vs 车载音频架构](#61-传统手机-vs-车载音频架构)
  - [6.2 什么是"硬件总线"](#62-什么是硬件总线)
  - [6.3 company 的 Bus 分配详解](#63-company-的-bus-分配详解)
  - [6.4 AudioBus 路由工作流程](#64-audiobus-路由工作流程)
  - [6.5 Duck 在多总线架构中的实现](#65-duck-在多总线架构中的实现)
  - [6.6 为什么需要多总线架构](#66-为什么需要多总线架构)
  - [6.7 TDM 时隙复用架构](#67-tdm-时隙复用架构)
  - [6.8 PCM 节点与 Bus 数量不对应的真相](#68-pcm-节点与-bus-数量不对应的真相)
- [七、完整调用链路](#七完整调用链路)
- [八、关键代码位置汇总](#八关键代码位置汇总)

---

## 一、AudioTrack 基础

### 1.1 AudioTrack 类定义

**头文件位置**: `frameworks/av/media/libaudioclient/include/media/AudioTrack.h`

```cpp
class AudioTrack : public AudioSystem::AudioDeviceCallback
{
    // Native 层音频播放轨道实现
    // 负责与 AudioFlinger 服务通信，管理音频数据的播放
};
```

**实现文件**: `frameworks/av/media/libaudioclient/AudioTrack.cpp` (3731 行)

### 1.2 核心功能

1. **构造函数和初始化** - 创建音频轨道，配置采样率、声道、格式等参数
2. **音频播放控制** - `start()`, `stop()`, `pause()`, `flush()` 等方法
3. **数据写入** - `write()` 方法用于向音频缓冲区写入 PCM 数据
4. **音量控制** - `setVolume()`, `getVolume()`
5. **播放位置查询** - `getPosition()`, `getPlaybackHeadPosition()`
6. **与 AudioFlinger 通信** - 通过 Binder 与系统音频服务交互

---

## 二、VolumeShaper 原理

### 2.1 VolumeShaper 概述

**定义位置**: `frameworks/av/include/media/VolumeShaper.h`

VolumeShaper 是 Android 音频系统的**音量渐变控制器**，可以实现：
- 音量淡入淡出（Fade In/Out）
- Duck 降音效果
- 自定义音量曲线

### 2.2 核心参数

```cpp
class VolumeShaper {
public:
    // 系统级 VolumeShaper 最大数量
    static const int kSystemVolumeShapersMax = 16;
    
    // 应用级 VolumeShaper 最大数量
    static const int kUserVolumeShapersMax = 16;
    
    // ID=1 保留给系统 Duck
    // "1" is reserved for system ducking.
};
```

**Configuration 核心属性**:
- **ID**: 唯一标识（0-15 系统级，16+ 应用级）
- **Curve**: 时间-音量曲线（times[], volumes[]）
- **Duration**: 渐变时长（毫秒）
- **InterpolatorType**: 插值方式
- **OptionFlags**: 选项标志（CLOCK_TIME, VOLUME_IN_DBFS）

### 2.3 插值类型

| 类型 | 常量值 | 效果 | 适用场景 |
|------|--------|------|----------|
| **STEP** | 0 | 阶跃变化，无渐变 | 测试、特殊效果 |
| **LINEAR** | 1 | 线性渐变，匀速变化 | 简单 Duck、快速响应 |
| **CUBIC** | 2 | 三次样条，平滑但可能过冲 | 音乐淡入淡出 |
| **CUBIC_MONOTONIC** | 3 | 单调三次样条，平滑且单调 | **推荐用于 Duck** |

---

## 三、系统 Duck 机制

### 3.1 什么是 Duck

**Duck（降音/压低音量）** 是车载音频系统的核心功能：当高优先级音频（如导航、语音助手）播放时，自动降低低优先级音频（如音乐）的音量，而不是完全暂停。

**典型场景**:
- 导航播报时，音乐音量自动降低 50%
- 语音助手唤醒时，媒体音量降低
- 通话结束时，音乐音量自动恢复

### 3.2 Duck 的两层实现

#### 第一层：AudioFocus 层（框架决策层）

**职责**: 决定**是否需要 Duck**，基于交互矩阵规则

**核心文件**:
- `packages/services/Car/service/src/com/android/car/audio/AudioMatrix.java` - 交互矩阵定义
- `packages/services/Car/service/src/com/android/car/audio/FocusInteraction.java` - 交互评估逻辑
- `packages/services/Car/service/src/com/android/car/audio/CarAudioFocus.java` - 焦点管理

#### 第二层：VolumeShaper 层（Native 执行层）

**职责**: **实际执行音量渐变**，通过 VolumeShaper ID=1 实现

**核心文件**:
- `frameworks/base/services/core/java/com/android/server/audio/PlaybackActivityMonitor.java` - DuckingManager

### 3.3 AudioMatrix 交互矩阵

**位置**: `packages/services/Car/service/src/com/android/car/audio/AudioMatrix.java`

**三种交互模式**:

```java
// FocusInteraction.java:58-63
static final int INTERACTION_REJECT = 0;      // 拒绝新请求
static final int INTERACTION_EXCLUSIVE = 1;   // 独占焦点（其他暂停）
static final int INTERACTION_CONCURRENT = 2;  // 并发（其他可 Duck）
```

**矩阵示例**（以音乐为焦点持有者）:

```java
/* Focus holder:USAGE_BT_AUDIO (BT音乐播放中) */
{
    INTERACTION_CONCURRENT, // USAGE_NAVI 导航 → 并发（音乐Duck）
    INTERACTION_CONCURRENT, // USAGE_VR 语音 → 并发（音乐Duck）
    INTERACTION_EXCLUSIVE,  // USAGE_CALL 电话 → 独占（音乐暂停）
    INTERACTION_CONCURRENT, // USAGE_CARPLAY_NAVI CarPlay导航 → Duck
    INTERACTION_EXCLUSIVE,  // USAGE_ECALL 紧急呼叫 → 独占（音乐暂停）
}
```

### 3.4 Duck 决策流程

**位置**: `FocusInteraction.java:206-220`

```java
case INTERACTION_CONCURRENT:
    // 判断是否需要 Duck
    final boolean allowDuckingHolder =
        (focusHolder.getAudioFocusInfo().getGainRequest() == 
         AudioManager.AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK);
    
    // 以下情况不 Duck，直接暂停（LOSS）
    if (!allowDucking && !allowDuckingHolder
        || focusHolder.wantsPauseInsteadOfDucking()  // 设置了 PAUSE_ON_DUCK
        || focusHolder.receivesDuckEvents()) {       // 有权限接收 Duck 事件
        focusLosers.add(focusHolder);  // 发送 LOSS_TRANSIENT
    }
    // 否则发送 LOSS_TRANSIENT_CAN_DUCK（允许 Duck）
    return AudioManager.AUDIOFOCUS_REQUEST_GRANTED;
```

**Duck vs 暂停的对比**:

| 特性 | Duck | 暂停（LOSS_TRANSIENT） |
|------|------|----------------------|
| **焦点类型** | `AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK` | `AUDIOFOCUS_GAIN_TRANSIENT` |
| **交互矩阵** | `INTERACTION_CONCURRENT` | `INTERACTION_EXCLUSIVE` |
| **音频状态** | 继续播放，音量降低 | 暂停播放 |
| **用户体验** | 背景音持续，感知自然 | 完全静音 |
| **适用场景** | 导航播报、通知音 | 电话、紧急呼叫、VR |
| **恢复速度** | 快速（只需恢复音量） | 较慢（需重新缓冲） |

---

## 四、VolumeShaper 触发机制

### 4.1 核心发现

**重要**: Duck 的触发不在 AudioPolicyService（Native 层），而是在 `AudioService`（Java 层）的 `PlaybackActivityMonitor` 中！

### 4.2 Duck 曲线配置

**位置**: `PlaybackActivityMonitor.java:58-68`

```java
/*package*/ static final int VOLUME_SHAPER_SYSTEM_DUCK_ID = 1;
/*package*/ static final int VOLUME_SHAPER_SYSTEM_FADEOUT_ID = 2;

private static final VolumeShaper.Configuration DUCK_VSHAPE =
    new VolumeShaper.Configuration.Builder()
        .setId(VOLUME_SHAPER_SYSTEM_DUCK_ID)  // ID = 1
        .setCurve(
            new float[] { 0.f, 1.f } /* times */,      // 时间轴：0% → 100%
            new float[] { 1.f, 0.2f } /* volumes */)   // 音量：100% → 20%
        .setOptionFlags(VolumeShaper.Configuration.OPTION_FLAG_CLOCK_TIME)
        .setDuration(MediaFocusControl.getFocusRampTimeMs(
            AudioManager.AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK,
            new AudioAttributes.Builder()
                .setUsage(AudioAttributes.USAGE_NOTIFICATION)
                .build()))
        .build();
```

**Duck 曲线效果**:
```
时间:   0ms ━━━━━━━━━━━━━→ rampTime (通常 1000ms)
音量:   1.0 ━━━━━━━━━━━→ 0.2 (降低到 20%)
        └──── 线性渐变 ────┘
```

### 4.3 完整触发流程

#### 步骤 1: AudioFocus 请求导致其他应用需要 Duck

```
导航应用请求焦点 (AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK)
    ↓
CarAudioFocus/MediaFocusControl 评估焦点请求
    ↓
查询 AudioMatrix → INTERACTION_CONCURRENT (允许 Duck)
    ↓
调用 PlaybackActivityMonitor.duckPlayersForFocus()
```

#### 步骤 2: DuckingManager.duckUid() 收集需要 Duck 的播放器

**位置**: `PlaybackActivityMonitor.java:856-865`

```java
synchronized void duckUid(int uid, ArrayList<AudioPlaybackConfiguration> apcsToDuck) {
    if (!mDuckers.containsKey(uid)) {
        mDuckers.put(uid, new DuckedApp(uid));
    }
    final DuckedApp da = mDuckers.get(uid);
    for (AudioPlaybackConfiguration apc : apcsToDuck) {
        da.addDuck(apc, false /*skipRamp*/);
    }
}
```

#### 步骤 3: addDuck() 调用 applyVolumeShaper

**位置**: `PlaybackActivityMonitor.java:923-938`

```java
void addDuck(@NonNull AudioPlaybackConfiguration apc, boolean skipRamp) {
    final int piid = new Integer(apc.getPlayerInterfaceId());
    if (mDuckedPlayers.contains(piid)) {
        return;  // 已经 Duck 过，跳过
    }
    
    // ★★★ 关键调用：应用 VolumeShaper ★★★
    apc.getPlayerProxy().applyVolumeShaper(
        DUCK_VSHAPE,                    // Duck 曲线配置
        skipRamp ? PLAY_SKIP_RAMP       // 如果跳过渐变，直接跳到终点
                 : PLAY_CREATE_IF_NEEDED // 否则创建并播放渐变
    );
    
    mDuckedPlayers.add(piid);
}
```

#### 步骤 4: 新播放器启动时检查是否需要 Duck

**位置**: `PlaybackActivityMonitor.java:877-885`

```java
synchronized void checkDuck(@NonNull AudioPlaybackConfiguration apc) {
    final DuckedApp da = mDuckers.get(apc.getClientUid());
    if (da == null) {
        return;
    }
    // 新播放器启动时直接跳到 Duck 状态（跳过渐变）
    da.addDuck(apc, true /*skipRamp*/);
}
```

### 4.4 取消 Duck 恢复音量

**位置**: `PlaybackActivityMonitor.java:940-962`

```java
void removeUnduckAll(HashMap<Integer, AudioPlaybackConfiguration> players) {
    for (int piid : mDuckedPlayers) {
        final AudioPlaybackConfiguration apc = players.get(piid);
        if (apc != null) {
            // ★★★ 关键调用：反转 VolumeShaper 恢复音量 ★★★
            apc.getPlayerProxy().applyVolumeShaper(
                DUCK_ID,                    // 仅指定 ID=1
                VolumeShaper.Operation.REVERSE  // 反转曲线（0.2 → 1.0）
            );
        }
    }
    mDuckedPlayers.clear();
}
```

**恢复原理**:
```
REVERSE 操作会反转原始曲线：
原始: 1.0 → 0.2 (Duck)
反转: 0.2 → 1.0 (恢复)

时间:   0ms ━━━━━━━━━━━━━→ rampTime
音量:   0.2 ━━━━━━━━━━━→ 1.0
        └──── 线性渐变 ────┘
```

---

## 五、自定义 Duck 曲线和时长

### 5.1 方案一：修改系统默认配置（推荐用于整机定制）

#### 修改 Duck 曲线形状

**文件**: `PlaybackActivityMonitor.java:58-68`

**示例 1: 慢速 Duck（1.5秒降到 30%）**

```java
private static final VolumeShaper.Configuration DUCK_VSHAPE =
    new VolumeShaper.Configuration.Builder()
        .setId(VOLUME_SHAPER_SYSTEM_DUCK_ID)
        .setCurve(
            new float[] { 0.f, 1.f },      // 时间轴
            new float[] { 1.f, 0.3f })     // 音量：100% → 30%（更温和）
        .setOptionFlags(VolumeShaper.Configuration.OPTION_FLAG_CLOCK_TIME)
        .setDuration(1500)  // 固定 1.5 秒
        .build();
```

**示例 2: 多段曲线（快速降低，缓慢恢复）**

```java
private static final VolumeShaper.Configuration DUCK_VSHAPE =
    new VolumeShaper.Configuration.Builder()
        .setId(VOLUME_SHAPER_SYSTEM_DUCK_ID)
        .setCurve(
            new float[] { 0.f, 0.2f, 0.8f, 1.f },  // 4个时间点
            new float[] { 1.f, 0.2f, 0.2f, 1.f })  // 快降→保持→慢升
        .setOptionFlags(VolumeShaper.Configuration.OPTION_FLAG_CLOCK_TIME)
        .setDuration(2000)
        .setInterpolatorType(VolumeShaper.Configuration.INTERPOLATOR_TYPE_CUBIC_MONOTONIC)
        .build();
```

效果：
```
音量: 1.0 ━┓
          ┃━━ 0.2 (20% 处快速降低)
          ┃━━━━━━━━━ 0.2 (保持 Duck 状态)
          ┗━━━━━━━━━┓ 1.0 (缓慢恢复到 100%)
时间:   0ms   400ms    1600ms   2000ms
```

#### 修改 Duck 时长

**文件**: `MediaFocusControl.java:823-850`

**按 Usage 精细控制**:

```java
protected static int getFocusRampTimeMs(int focusGain, AudioAttributes attr) {
    // 根据 company 项目需求定制
    switch (attr.getUsage()) {
        // 导航相关 - 快速响应
        case AudioAttributes.USAGE_ASSISTANCE_NAVIGATION_GUIDANCE:
            return 300;   // 本地导航：300ms
        case CarAudioAttributes.USAGE_CARPLAY_NAVI:
            return 350;   // CarPlay 导航：350ms
        case CarAudioAttributes.USAGE_HICAR_NAVI:
            return 350;   // HiCar 导航：350ms
            
        // 语音相关 - 中等速度
        case AudioAttributes.USAGE_ASSISTANT:
        case CarAudioAttributes.USAGE_VR:
            return 500;   // 语音助手：500ms
            
        // 电话相关 - 较慢
        case AudioAttributes.USAGE_VOICE_COMMUNICATION:
        case CarAudioAttributes.USAGE_CALL:
            return 600;   // 通话：600ms
            
        // 音乐/媒体 - 最慢
        case AudioAttributes.USAGE_MEDIA:
        case CarAudioAttributes.USAGE_BT_AUDIO:
            return 1200;  // 媒体：1.2秒
            
        default:
            return 500;   // 默认 500ms
    }
}
```

### 5.2 方案二：车载系统定制（company 项目推荐）

创建配置文件 `vendor/company/build/audio/duck_config.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<duckConfiguration>
    <default>
        <duration>800</duration>
        <minVolume>0.25</minVolume>
        <interpolator>CUBIC_MONOTONIC</interpolator>
    </default>
    
    <usageRules>
        <usage name="USAGE_NAVI">
            <duration>400</duration>
            <minVolume>0.3</minVolume>
        </usage>
        <usage name="USAGE_VR">
            <duration>500</duration>
            <minVolume>0.2</minVolume>
        </usage>
    </usageRules>
</duckConfiguration>
```

### 5.3 方案三：运行时动态调整（无需重启）

通过 SystemProperties 控制：

```java
private static VolumeShaper.Configuration createDuckConfiguration() {
    int duration = SystemProperties.getInt("persist.audio.duck.duration", 0);
    float minVolume = SystemProperties.getFloat("persist.audio.duck.minvolume", 0.2f);
    
    if (duration == 0) {
        duration = MediaFocusControl.getFocusRampTimeMs(...);
    }
    
    return new VolumeShaper.Configuration.Builder()
        .setId(VOLUME_SHAPER_SYSTEM_DUCK_ID)
        .setCurve(new float[] { 0.f, 1.f }, new float[] { 1.f, minVolume })
        .setDuration(duration)
        .build();
}
```

**使用方式**:

```bash
# 修改 Duck 时长为 600ms
adb shell setprop persist.audio.duck.duration 600

# 修改最低音量为 30%
adb shell setprop persist.audio.duck.minvolume 0.3

# 重启 AudioService 生效
adb shell stop; adb shell start
```

### 5.4 方案四：应用层自定义 Duck

```java
// 在音乐 App 中创建自定义 VolumeShaper
VolumeShaper.Configuration myDuckConfig = 
    new VolumeShaper.Configuration.Builder()
        .setId(100)  // 应用级 ID（>=16）
        .setCurve(
            new float[] { 0.f, 0.1f, 0.9f, 1.f },
            new float[] { 1.f, 0.1f, 0.1f, 1.f })
        .setDuration(2000)
        .build();

audioManager.requestAudioFocus(new AudioManager.OnAudioFocusChangeListener() {
    @Override
    public void onAudioFocusChange(int focusChange) {
        switch (focusChange) {
            case AudioManager.AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK:
                audioTrack.applyVolumeShaper(myDuckConfig, play);
                break;
            case AudioManager.AUDIOFOCUS_GAIN:
                audioTrack.applyVolumeShaper(
                    new VolumeShaper.Configuration(100),
                    VolumeShaper.Operation.REVERSE);
                break;
        }
    }
}, AudioManager.AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK);
```

---

## 六、company 多 AudioBus 硬件路由架构

### 6.1 传统手机 vs 车载音频架构

#### 传统手机架构（单总线混音）

```
┌─────────────────────────────────────────────┐
│              Android AudioService            │
└──────────────┬──────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────┐
│          AudioFlinger (混音器)               │
│                                             │
│  ┌──────┐  ┌──────┐  ┌──────┐              │
│  │音乐   │+ │导航   │+ │VR    │ = 混合后的音频│
│  └──────┘  └──────┘  └──────┘              │
└──────────────┬──────────────────────────────┘
               │
               ▼ (单一 PCM 数据流)
┌─────────────────────────────────────────────┐
│          Audio HAL (音频驱动)                │
└──────────────┬──────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────┐
│          扬声器 (Stereo)                     │
└─────────────────────────────────────────────┘
```

**特点**：所有音频在软件层混音，走同一个输出通道

---

#### company 车载架构（多总线独立路由）

**配置文件映射**：

```xml
<!-- audio_policy_configuration.xml:55-65 -->
<attachedDevices>
    <item>Media Bus</item>        ← bus0 (音乐)
    <item>Nav Guidance Bus</item> ← bus1 (导航)
    <item>VR TTS Bus</item>       ← bus2 (语音)
    <item>Phone Bus</item>        ← bus3 (电话)
    <item>MVC APS Bus</item>      ← bus4 (安全警告)
    <item>Beep</item>             ← bus16 (系统提示音)
</attachedDevices>
```

```
┌──────────────────────────────────────────────────────────────┐
│                    Android AudioService                       │
└──┬────────┬────────┬────────┬────────┬───────────────────────┘
   │        │        │        │        │
   ▼        ▼        ▼        ▼        ▼
┌──────────────────────────────────────────────────────────────┐
│                  AudioFlinger (路由层)                        │
│                                                              │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐          │
│  │Media │  │Nav   │  │VR TTS│  │Phone │  │MVC   │          │
│  │ bus0 │  │ bus1 │  │ bus2 │  │ bus3 │  │ bus4 │          │
│  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘          │
└─────┼─────────┼─────────┼─────────┼─────────┼────────────────┘
      │         │         │         │         │
      ▼         ▼         ▼         ▼         ▼
┌──────────────────────────────────────────────────────────────┐
│                   Audio HAL (多通道输出)                      │
│                                                              │
│  bus0      bus1      bus2      bus3      bus4               │
│   ↓         ↓         ↓         ↓         ↓                 │
└───┼─────────┼─────────┼─────────┼─────────┼──────────────────┘
    │         │         │         │         │
    ▼         ▼         ▼         ▼         ▼
┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
│DSP混音│  │DSP混音│  │DSP混音│  │DSP混音│  │DSP混音│
│或独立 │  │或独立 │  │或独立 │  │或独立 │  │或独立 │
│放大器 │  │放大器 │  │放大器 │  │放大器 │  │放大器 │
└──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘
   │         │         │         │         │
   ▼         ▼         ▼         ▼         ▼
┌──────────────────────────────────────────────────────────────┐
│                    硬件音频路由矩阵                            │
│                    (DSP/音频功放芯片)                          │
│                                                              │
│  可以根据车速、场景、区域独立控制每个总线的：                    │
│  - 音量（Active Volume）                                      │
│  - 均衡器（EQ）                                               │
│  - 声像（Pan/平衡）                                           │
│  - 输出扬声器组合                                             │
└──────────────┬───────────────────────────────────────────────┘
               │
               ▼
    ┌──────────────────────┐
    │   车内扬声器阵列       │
    │                      │
    │  前左  前右  中置     │
    │  后左  后右  低音炮   │
    └──────────────────────┘
```

### 6.2 什么是"硬件总线"

在 company 车载系统中，**"Bus" 不是指物理的 I2C/SPI 总线**，而是指：

**逻辑音频通道 → DSP 硬件路由 → 独立处理 → 独立输出**

从配置文件看：

```xml
<!-- audio_policy_configuration.xml:110-127 -->

<!-- bus0: 音乐总线 - 立体声 -->
<devicePort tagName="Media Bus" role="sink" type="AUDIO_DEVICE_OUT_BUS"
        address="bus0">
    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
             samplingRates="48000" channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
</devicePort>

<!-- bus1: 导航总线 - 单声道 -->
<devicePort tagName="Nav Guidance Bus" role="sink" type="AUDIO_DEVICE_OUT_BUS"
        address="bus1">
    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
             samplingRates="48000" channelMasks="AUDIO_CHANNEL_OUT_MONO"/>
</devicePort>

<!-- bus2: VR总线 - 单声道 -->
<devicePort tagName="VR TTS Bus" role="sink" type="AUDIO_DEVICE_OUT_BUS"
        address="bus2">
    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
             samplingRates="48000" channelMasks="AUDIO_CHANNEL_OUT_MONO"/>
</devicePort>
```

**关键差异**：
- **bus0 (音乐)**: 立体声 (`AUDIO_CHANNEL_OUT_STEREO`) - 高质量
- **bus1 (导航)**: 单声道 (`AUDIO_CHANNEL_OUT_MONO`) - 语音清晰
- **bus2 (VR)**: 单声道 (`AUDIO_CHANNEL_OUT_MONO`) - 语音识别

#### 硬件层面的实现

这些 Bus 在硬件上对应：

```
Android 系统层
     │
     ├── bus0 (PCM 数据流 0) ──→ DSP 输入通道 0 ──→ 混音器 A ──→ 前左/前右扬声器
     │
     ├── bus1 (PCM 数据流 1) ──→ DSP 输入通道 1 ──→ 混音器 B ──→ 全车扬声器（中置优先）
     │
     ├── bus2 (PCM 数据流 2) ──→ DSP 输入通道 2 ──→ 混音器 C ──→ 驾驶员侧扬声器
     │
     ├── bus3 (PCM 数据流 3) ──→ DSP 输入通道 3 ──→ 混音器 D ──→ 全车扬声器
     │
     └── bus4 (PCM 数据流 4) ──→ DSP 输入通道 4 ──→ 混音器 E ──→ 警告音专用扬声器
```

**硬件优势**：
1. **独立音量控制**：每个 Bus 有独立的硬件音量控制（配置中的 `<gains>` 标签）
2. **独立 EQ 处理**：导航语音可以增强人声频段，音乐保持全频段
3. **独立声场定位**：导航可以从驾驶位扬声器播放，音乐全车播放
4. **硬件优先级**：紧急警告（bus4）可以硬件级打断其他音频

### 6.3 company 的 Bus 分配详解

根据配置文件，完整的 Bus 分配：

```xml
<!-- car_audio_configuration.xml:35-89 -->

bus16: Beep (系统提示音)          - 0~3 步
  └─ context: system_sound

bus4:  MVC AP音声 (安全警告)       - 1~40 步
  └─ context: safety

bus1: 导航 (Navi/CarPlay Alternate) - 0~11 步 (Active Volume)
  ├─ context: navigation
  └─ 支持主动音量控制（根据车速自动调整）

bus2: 语音识别 (VR)                - 1~11 步
  ├─ CarLife VR
  ├─ Local VR
  ├─ SIRI
  └─ 天猫精灵

bus0: 音乐/媒体                    - 0~40 步
  ├─ context: music (第三方音乐)
  ├─ context: diag (诊断)
  └─ context: boot (开机动画音频)

bus3: 电话/铃声                    - 0~40 步
  ├─ context: call (语音通话)
  ├─ context: ring (CarPlay/HiCar 铃声)
  ├─ context: local_ring (本地铃声)
  └─ context: call_ring (带内铃声)
```

### 6.4 AudioBus 路由工作流程

#### 场景：导航播报时的音频路由

```
[高德导航 App] 开始播报"前方 500 米右转"
    ↓
1. AudioAttributes 设置
   AudioAttributes.Builder()
       .setUsage(USAGE_ASSISTANCE_NAVIGATION_GUIDANCE)
       .build()
    ↓
2. CarAudioService 查找配置
   car_audio_configuration.xml:
   <context context="navigation"/>
       → <device address="bus1">
    ↓
3. AudioPolicyService 路由决策
   选择输出设备: Nav Guidance Bus (bus1)
    ↓
4. AudioTrack 创建
   audio_track_cblk_t 分配缓冲区
   绑定到 bus1 混音端口 (nav_guidance mixPort)
    ↓
5. AudioFlinger 路由
   route type="mix" sink="Nav Guidance Bus" sources="nav_guidance"
    ↓
6. Audio HAL 处理
   将 PCM 数据写入 bus1 设备节点
   /dev/snd/pcmC0D1p (假设)
    ↓
7. DSP/硬件处理
   ┌──────────────────────────────────┐
   │ bus1 独立处理链：                  │
   │ 1. 硬件音量控制 (0~11 Step)       │
   │ 2. EQ: 增强 300Hz-3kHz (人声)    │
   │ 3. 声场: 中置扬声器优先            │
   │ 4. 主动音量控制 (根据车速调整)     │
   └──────────────────────────────────┘
    ↓
8. 扬声器输出
   中置扬声器：导航语音（大音量）
   前后扬声器：音乐（Duck 后的背景音）
```

**同时**，音乐播放不受影响：

```
[QQ音乐 App] 继续播放
    ↓
AudioAttributes.USAGE_MEDIA
    ↓
car_audio_configuration.xml:
<context context="music"/>
    → <device address="bus0">
    ↓
AudioFlinger 路由到 Media Bus (bus0)
    ↓
DSP 对 bus0 应用 Duck 降音（通过 VolumeShaper）
    ↓
bus0 音量降到 25%
```

### 6.5 Duck 在多总线架构中的实现

**重要**：虽然导航和音乐走不同的硬件 Bus，但 **Duck 仍然在软件层实现**！

```
┌─────────────────────────────────────────────────────┐
│                  AudioService                        │
│                                                      │
│  PlaybackActivityMonitor.DuckingManager              │
│  ↓                                                   │
│  检测到 bus1 (导航) 播放                              │
│  ↓                                                   │
│  对 bus0 (音乐) 应用 VolumeShaper                    │
│  apc.getPlayerProxy().applyVolumeShaper(             │
│      DUCK_VSHAPE,  // 1.0 → 0.2                     │
│      PLAY_CREATE_IF_NEEDED                           │
│  )                                                   │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│                  AudioFlinger                        │
│                                                      │
│  bus0 (音乐) 的 Track:                               │
│  volume *= shaperVolume(0.2)  ← Duck 在这里生效      │
│                                                      │
│  bus1 (导航) 的 Track:                               │
│  volume *= 1.0  ← 不受影响                           │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│                  Audio HAL                           │
│                                                      │
│  bus0 → DSP → 前左/前右 (音量 25%)                  │
│  bus1 → DSP → 中置优先 (音量 100%)                  │
└─────────────────────────────────────────────────────┘
```

**所以 Duck 的本质是**：
- **决策层**: CarAudioService (AudioMatrix) 决定是否需要 Duck
- **执行层**: AudioService (DuckingManager) 对音乐 Bus 应用 VolumeShaper
- **效果**: 音乐 Bus (bus0) 的软件音量降低，导航 Bus (bus1) 不受影响
- **硬件层**: DSP 接收到的 bus0 数据已经是降低音量后的 PCM

### 6.6 为什么需要多总线架构

#### 1. 硬件级主动音量控制 (Active Volume)

```xml
<!-- bus1 支持 Active Volume -->
<group><!-- C: Navi - Active Volume 1~11 Step -->
    <device address="bus1">
        <context context="navigation"/>
    </device>
</group>
```

**Active Volume 工作原理**：
```
车速传感器 ──→ DSP ──→ 自动调整 bus1 音量
  │
  ├── 0 km/h:   音量 Step 3
  ├── 60 km/h:  音量 Step 6
  ├── 100 km/h: 音量 Step 9
  └── 120 km/h: 音量 Step 11
```

**为什么不用软件 Duck？**
- 软件 Duck 响应慢（需要 AudioService 处理）
- 硬件 Active Volume 实时响应（DSP 直接读取车速 CAN 总线）
- 不依赖 Android 系统（即使系统卡顿也不影响）

#### 2. 独立声场定位

```
导航播报时：
- bus1 (导航) → DSP 路由 → 中置扬声器 + 前左/前右（小音量）
- bus0 (音乐) → DSP 路由 → 全车扬声器（Duck 后）

电话接听时：
- bus3 (电话) → DSP 路由 → 驾驶员侧扬声器（隐私）
- bus0 (音乐) → DSP 路由 → 后排扬声器（乘客继续听）
```

#### 3. 硬件优先级打断

```
bus4 (MVC AP 安全警告) 播放时：
DSP 硬件级操作：
1. 立即 mute bus0, bus1, bus2, bus3
2. bus4 以最大音量播放到全车扬声器
3. 播放结束后自动恢复

无需 Android 系统参与！
```

#### 4. 与传统混音的对比

| 特性 | 传统手机（单总线） | company 车载（多总线） |
|------|------------------|-------------------|
| **混音位置** | Android 软件层 (AudioFlinger) | DSP/硬件层 |
| **数据流** | 所有音频混合成单一 PCM | 每个 Bus 独立 PCM 数据流 |
| **音量控制** | 软件统一控制 | 每个 Bus 独立硬件音量 |
| **EQ 处理** | 统一 EQ | 每个 Bus 独立 EQ |
| **声场定位** | 无法区分 | 可独立控制输出扬声器 |
| **延迟** | 软件混音延迟 | 硬件直通，低延迟 |
| **优先级** | 软件控制 | 硬件级优先级 |
| **音质** | 多次重采样可能损失 | 独立处理，无互相干扰 |

### 6.7 TDM 时隙复用架构

#### 核心发现

从配置文件 [`audio_platform_info_ms.xml`](file:///home/kongchaochao/work/3QK/buildsystem/android12/vendor/company/build/audio/audio_platform_info_ms.xml) 可以看到：

```xml
<!-- audio_platform_info_ms.xml:4-8 -->
<card name="x9msmach">
    <playback bit_width="16" rate="48000" channels="8" bind_bus_id="0,2,3,5,8" />
    <capture bit_width="16" rate="48000" channels="4" />
</card>
```

**关键信息**：
- **1 个 Sound Card** (x9msmach)
- **8 个音频通道** (channels="8")
- **绑定 5 个 Bus** (bind_bus_id="0,2,3,5,8")
- 但有 **6+ 个逻辑 Bus** (bus0, bus1, bus2, bus3, bus4, bus16)

#### TDM 工作原理

company 使用的是 **TDM (Time Division Multiplexing)** 时隙复用技术，这是 PCM 节点和 Bus 不对应的核心原因！

```
一个 PCM 设备 = 一个 ALSA PCM 节点 = /dev/snd/pcmC0D0p

TDM 帧结构 (8 通道, 48kHz, 16bit):
┌────────────────────────────────────────────────────────┐
│ TDM Frame (1 帧 = 8 个时隙)                             │
│                                                        │
│ Slot 0 │ Slot 1 │ Slot 2 │ Slot 3 │ ... │ Slot 7     │
│ bus0   │ bus0   │ bus0   │ bus0   │     │ bus3       │
│ (L)    │ (R)    │ (L)    │ (R)    │     │ (Mono)     │
└────────────────────────────────────────────────────────┘

一个 PCM 数据流包含多个时隙 (Slots)
每个时隙可以映射到不同的逻辑 Bus
```

#### Slot 映射配置

**位置**: [`audio_platform_info_ms.xml:25-42`](file:///home/kongchaochao/work/3QK/buildsystem/android12/vendor/company/build/audio/audio_platform_info_ms.xml#L25-L42)

```xml
<slot_map enable="true">
    <slot_map_card name="x9msmach">
        <!-- media: bus0 占用 Slot 0,1,2,3 (4通道立体声) -->
        <out bus0_slot="0,1,2,3" type="mix" />
        
        <!-- phone ring: bus2 占用 Slot 0 -->
        <out bus2_slot="0" type="mix" />
        
        <!-- navi: bus3 占用 Slot 6,7 (2通道单声道) -->
        <out bus3_slot="6,7" type="mix" />
        
        <!-- system: bus5 占用 Slot 0,1,2,3 -->
        <out bus5_slot="0,1,2,3" type="mix" />
        
        <!-- phone call: bus8 占用 Slot 4 (直通模式) -->
        <out bus8_slot="4" type="direct" />
        
        <!-- 麦克风输入 -->
        <in build_in_mic="0,1"/>
        <in phone_ext_mic="0,1,2,3"/>
    </slot_map_card>
</slot_map>
```

### 6.8 PCM 节点与 Bus 数量不对应的真相

#### 实际的硬件映射

```
Android 逻辑 Bus          ALSA PCM 节点         TDM 时隙
────────────────────────────────────────────────────────

bus0 (Music)         ──→   pcmC0D0p (设备 0)   ──→   Slot 0,1,2,3
bus1 (Navi)          ──→   pcmC0D0p (设备 0)   ──→   Slot 6,7  ← 注意！
bus2 (VR)            ──→   pcmC0D0p (设备 0)   ──→   Slot ?
bus3 (Phone)         ──→   pcmC0D0p (设备 0)   ──→   Slot 4
bus4 (Safety)        ──→   pcmC0D0p (设备 0)   ──→   Slot ?
bus16 (Beep)         ──→   pcmC0D0p (设备 0)   ──→   Slot ?

所有 Bus 共用同一个 PCM 设备！
通过 TDM 时隙区分不同的音频流
```

#### 为什么会这样设计？

**1. 硬件限制**
```
DSP 芯片 (如 NXP SAF775D/SAF775DS) 
    ↓
提供 1 个 TDM 接口
    ↓
TDM 接口支持 8 个时隙 (Slot 0-7)
    ↓
每个时隙可以独立路由到不同扬声器
```

**2. 软件抽象**
```
AudioFlinger 层：
    ├── 看到 6 个独立的 Bus (bus0, bus1, bus2, bus3, bus4, bus16)
    ├── 每个 Bus 可以独立控制音量、路由
    └─→ 对上层应用透明

Audio HAL 层：
    ├── 打开 1 个 PCM 设备 (/dev/snd/pcmC0D0p)
    ├── 配置 TDM 格式 (8 通道)
    └─→ 将所有 Bus 的数据打包到一个 TDM 帧

DSP 硬件层：
    ├── 接收 TDM 数据流
    ├── 根据 Slot 分离不同音频
    └─→ 独立处理每个 Slot (EQ、音量、路由)
```

#### 完整的音频数据流

**场景：同时播放音乐和导航**

```
[Android 应用层]
    ├── QQ音乐 → bus0 → 立体声 PCM 数据
    └─→ 高德导航 → bus1 → 单声道 PCM 数据

[AudioFlinger 层]
    ├── bus0 的 Track → 混音端口 "media"
    └─→ bus1 的 Track → 混音端口 "nav_guidance"

[Audio HAL 层] (tinyalsa_hal.c)
    ↓
    打开 PCM 设备: pcm_open(card=0, device=0, PCM_OUT)
    ↓
    配置 TDM 格式:
        channels = 8
        format = PCM_FORMAT_S16_LE
        rate = 48000
    ↓
    打包数据到 TDM 帧:
    ┌──────────────────────────────────────────┐
    │ 写入一个 TDM Frame (8 个 Slot × 16bit)   │
    │                                          │
    │ Slot 0: bus0 左声道 (音乐 L)             │
    │ Slot 1: bus0 右声道 (音乐 R)             │
    │ Slot 2: bus0 左声道 (音乐 L)             │
    │ Slot 3: bus0 右声道 (音乐 R)             │
    │ Slot 4: bus3 (电话)                      │
    │ Slot 5: (空)                             │
    │ Slot 6: bus1 左声道 (导航)               │
    │ Slot 7: bus1 右声道 (导航)               │
    └──────────────────────────────────────────┘
    ↓
    写入 PCM 设备: pcm_write(pcm, tdm_frame, frame_size)

[DSP 硬件层] (SAF775D)
    ↓
    接收 TDM 数据流
    ↓
    根据 Slot 分离:
        Slot 0-3 → 混音器 A (音乐) → 前左/前右扬声器
        Slot 6-7 → 混音器 B (导航) → 中置扬声器
    ↓
    独立处理:
        音乐: 全频段 EQ, 音量 Step 25
        导航: 人声增强 EQ, 音量 Step 8 (Active Volume)
```

#### Audio Policy 配置 vs 实际硬件

**audio_policy_configuration.xml** (给 Android 框架看的):
```xml
<!-- 定义了 6 个逻辑 Bus -->
<devicePort tagName="Media Bus"      address="bus0">  ← 逻辑标识
<devicePort tagName="Nav Guidance Bus" address="bus1">  ← 逻辑标识
<devicePort tagName="VR TTS Bus"     address="bus2">  ← 逻辑标识
<devicePort tagName="Phone Bus"      address="bus3">  ← 逻辑标识
<devicePort tagName="MVC APS Bus"    address="bus4">  ← 逻辑标识
<devicePort tagName="Beep"           address="bus16"> ← 逻辑标识
```

**audio_platform_info_ms.xml** (给 Audio HAL 看的):
```xml
<!-- 定义了 5 个实际绑定的 Bus ID -->
<playback bind_bus_id="0,2,3,5,8" />
                              ↑ ↑ ↑ ↑ ↑
                              │ │ │ │ └─ bus8 (Phone Call)
                              │ │ │ └─── bus5 (System/Beep)
                              │ │ └───── bus3 (Navi) ← 注意编号不一致！
                              │ └─────── bus2 (VR)
                              └───────── bus0 (Music)
```

**原因**:
1. **Audio Policy 的 Bus 编号**是给 CarAudioService 用的**逻辑 ID**
2. **Audio HAL 的 Bus 编号**是底层硬件的**物理 Slot 映射**
3. 两者通过 **Audio HAL 的路由表** 进行转换

#### Audio HAL 的路由转换

**位置**: `tinyalsa_hal.c` 和 `audio_platform_info_ms.xml`

```c
// AudioFlinger 请求播放到 "Nav Guidance Bus" (bus1)
    ↓
Audio HAL 查找路由表:
    if (device == AUDIO_DEVICE_OUT_BUS && address == "bus1") {
        // 映射到 TDM Slot 6,7
        actual_bus_id = 3;  // audio_platform_info 中的 bus3
        slot_mask = (1 << 6) | (1 << 7);
    }
    ↓
打开 PCM 设备 (如果未打开):
    pcm = pcm_open(card=0, device=0, PCM_OUT, &config);
    config.channels = 8;  // TDM 8 通道
    ↓
将导航音频写入 Slot 6,7:
    tdm_frame[slot_6] = nav_sample_L;
    tdm_frame[slot_7] = nav_sample_R;
    pcm_write(pcm, tdm_frame, frame_size);
```

#### 总结：为什么 PCM 节点数量和 Bus 数量不对应？

| 项目 | 数量 | 说明 |
|------|------|------|
| **逻辑 Bus (Audio Policy)** | 6+ 个 | bus0, bus1, bus2, bus3, bus4, bus16... |
| **PCM 设备 (ALSA)** | 1-2 个 | pcmC0D0p (主卡), pcmC1D0p (参考卡) |
| **TDM 时隙** | 8 个 | Slot 0-7 |
| **绑定的 Bus ID** | 5 个 | 0, 2, 3, 5, 8 |

**根本原因**：

1. ✅ **TDM 时隙复用**：1 个 PCM 设备承载多个逻辑 Bus
2. ✅ **Slot 映射机制**：不同 Bus 占用不同的 TDM 时隙
3. ✅ **软件抽象层**：Audio Policy 看到的是逻辑 Bus，Audio HAL 处理的是物理 Slot
4. ✅ **DSP 硬件能力**：SAF775D 等车载 DSP 原生支持 TDM 多通道

**优势**：
- 🎯 减少 PCM 设备数量（节省内核资源）
- 🎯 降低延迟（一次 DMA 传输所有通道）
- 🎯 硬件级隔离（每个 Slot 独立处理）
- 🎯 灵活的路由（软件可重新映射 Slot）

---

## 七、完整调用链路

```
[导航应用] 请求 AudioFocus (AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK)
    ↓
[CarAudioFocus/MediaFocusControl] 评估焦点请求
    ↓
查询 AudioMatrix → INTERACTION_CONCURRENT (允许并发+Duck)
    ↓
┌─────────────────────────────┐
│ 通知音乐应用:                │
│ onAudioFocusChange(         │
│   LOSS_TRANSIENT_CAN_DUCK)  │
└─────────────────────────────┘
    ↓ 同时
[PlaybackActivityMonitor].duckPlayersForFocus()
    ↓
[DuckingManager].duckUid(uid, apcsToDuck)
    ↓
[DuckedApp].addDuck(apc, false)
    ↓
★★★ apc.getPlayerProxy().applyVolumeShaper(
        DUCK_VSHAPE,              // ID=1, curve=[(0,1.0),(1,0.2)]
        PLAY_CREATE_IF_NEEDED
    ) ★★★
    ↓
[AudioTrack.java] native_applyVolumeShaper()
    ↓
[JNI] android_media_AudioTrack.cpp
    ↓
[Native AudioTrack] applyVolumeShaper()
    ↓
[Binder] IAudioTrack → AudioFlinger
    ↓
[AudioFlinger::Track] mVolumeHandler->applyVolumeShaper()
    ↓
[FastMixer] volume *= shaperVolume(0.2)
    ↓
音乐音量降低到 20% ← ← ← ← ← ← ← ← ←
    
    
    
[导航播报结束]
    ↓
[PlaybackActivityMonitor].restoreVShapedPlayers()
    ↓
[DuckingManager].unduckUid()
    ↓
[DuckedApp].removeUnduckAll()
    ↓
★★★ apc.getPlayerProxy().applyVolumeShaper(
        DUCK_ID,                  // ID=1
        VolumeShaper.Operation.REVERSE
    ) ★★★
    ↓
音量从 0.2 恢复到 1.0
```

---

## 八、关键代码位置汇总

### 1. AudioTrack 层

| 文件 | 行号 | 功能 |
|------|------|------|
| `frameworks/av/media/libaudioclient/include/media/AudioTrack.h` | 862-864 | `applyVolumeShaper()` 声明 |
| `frameworks/av/media/libaudioclient/AudioTrack.cpp` | 2875-2905 | `applyVolumeShaper()` 实现 |

### 2. VolumeShaper 层

| 文件 | 行号 | 功能 |
|------|------|------|
| `frameworks/av/include/media/VolumeShaper.h` | 56-1150 | VolumeShaper 类定义 |
| `frameworks/av/include/media/VolumeShaper.h` | 70-76 | `kSystemVolumeShapersMax = 16`，ID=1 保留给 Duck |
| `frameworks/base/media/java/android/media/VolumeShaper.java` | 全文 | Java 层 VolumeShaper API |

### 3. Duck 决策层（CarAudio）

| 文件 | 行号 | 功能 |
|------|------|------|
| `packages/services/Car/service/src/com/android/car/audio/AudioMatrix.java` | 21-83 | 交互矩阵定义 |
| `packages/services/Car/service/src/com/android/car/audio/FocusInteraction.java` | 58-63 | 三种交互模式定义 |
| `packages/services/Car/service/src/com/android/car/audio/FocusInteraction.java` | 182-221 | Duck 决策逻辑 |
| `packages/services/Car/service/src/com/android/car/audio/CarAudioFocus.java` | 319-332 | 发送 Duck/LOSS 事件 |

### 4. Duck 执行层（AudioService）

| 文件 | 行号 | 功能 |
|------|------|------|
| `frameworks/base/services/core/java/com/android/server/audio/PlaybackActivityMonitor.java` | 55-56 | VolumeShaper ID 定义 |
| `frameworks/base/services/core/java/com/android/server/audio/PlaybackActivityMonitor.java` | 58-68 | **DUCK_VSHAPE 曲线配置** |
| `frameworks/base/services/core/java/com/android/server/audio/PlaybackActivityMonitor.java` | 853 | **class DuckingManager** |
| `frameworks/base/services/core/java/com/android/server/audio/PlaybackActivityMonitor.java` | 856-865 | `duckUid()` 触发 Duck |
| `frameworks/base/services/core/java/com/android/server/audio/PlaybackActivityMonitor.java` | 923-938 | **`addDuck()` 调用 applyVolumeShaper** |
| `frameworks/base/services/core/java/com/android/server/audio/PlaybackActivityMonitor.java` | 940-962 | `removeUnduckAll()` 取消 Duck |
| `frameworks/base/services/core/java/com/android/server/audio/MediaFocusControl.java` | 823-850 | `getFocusRampTimeMs()` 时长计算 |

### 5. 混音应用层（AudioFlinger）

| 文件 | 行号 | 功能 |
|------|------|------|
| `frameworks/av/services/audioflinger/Tracks.cpp` | 1353-1386 | Track::applyVolumeShaper() |
| `frameworks/av/services/audioflinger/Threads.cpp` | 5208-5210 | FastMixer 应用音量因子 |
| `frameworks/av/services/audioflinger/Threads.cpp` | 5912-5920 | DirectOutput 应用音量因子 |

---

## 附录：company 项目推荐配置

基于车载项目特点，推荐以下 Duck 配置：

**文件**: `PlaybackActivityMonitor.java`

```java
private static final VolumeShaper.Configuration DUCK_VSHAPE =
    new VolumeShaper.Configuration.Builder()
        .setId(VOLUME_SHAPER_SYSTEM_DUCK_ID)
        .setCurve(
            new float[] { 0.f, 0.15f, 1.f },    // 快速降低，缓慢恢复
            new float[] { 1.f, 0.25f, 1.f })    // 降到 25%，然后恢复
        .setOptionFlags(VolumeShaper.Configuration.OPTION_FLAG_CLOCK_TIME)
        .setDuration(1000)                       // 固定 1 秒
        .setInterpolatorType(
            VolumeShaper.Configuration.INTERPOLATOR_TYPE_CUBIC_MONOTONIC)
        .build();
```

**效果**:
- 前 150ms 快速降到 25%（导航立即清晰）
- 后 850ms 缓慢恢复到 100%（自然过渡）
- 使用单调三次插值（避免音量波动）

---
