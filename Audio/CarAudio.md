# Car Audio 架构关系图

## 系统架构总览

```
┌─────────────────────────────────────────────────────────────┐
│                      应用层 (App Layer)                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ 媒体应用  │  │ 导航应用  │  │ 电话应用  │  │ 语音助手  │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
└───────┼─────────────┼─────────────┼─────────────┼──────────┘
        │             │             │             │
        └─────────────┴─────────────┴─────────────┘
                      │ Binder IPC
┌─────────────────────┼───────────────────────────────────────┐
│              Car服务层 (Car Service Layer)                   │
│                     ▼                                        │
│  ┌────────────────────────────────────────────────────┐     │
│  │          CarAudioService (主服务)                    │     │
│  │  - 实现 ICarAudio.Stub                              │     │
│  │  - 实现 CarServiceBase                              │     │
│  │  - 2907行代码                                       │     │
│  └────┬──────────┬──────────┬──────────┬──────────────┘     │
│       │          │          │          │                     │
│  ┌────▼─────┐ ┌──▼──────┐ ┌▼────────┐│┌───────────────┐    │
│  │A2bService│ │AscServic│ │其他服务  │││CarServiceBase │    │
│  │          │ │e        │ │          │││               │    │
│  │-IA2b.Stub│ │-IAsc    │ │          │││- init()       │    │
│  │-转向提示  │ │-Stub    │ │          │││- release()    │    │
│  │-空间音频  │ │-声音增强│ │          │││- dump()       │    │
│  └──────────┘ └─────────┘ └──────────┘│└───────────────┘    │
└────────────────────────────────────────┼────────────────────┘
                                         │
┌────────────────────────────────────────┼────────────────────┐
│           音频管理层 (Audio Management)                     │
│                                        │                    │
│  ┌─────────────────────────────────────▼───────────────┐   │
│  │              CarAudioZone (音频区域)                  │   │
│  │  - Zone 0 (主驾驶区)                                 │   │
│  │  - Zone 1 (副驾驶区)                                 │   │
│  │  - Zone 2 (后排区)                                   │   │
│  └────┬──────────────┬──────────────┬─────────────────┘   │
│       │              │              │                      │
│  ┌────▼────────┐ ┌───▼────────┐ ┌───▼────────┐           │
│  │CarVolume    │ │CarVolume   │ │CarVolume   │           │
│  │Group 0      │ │Group 1     │ │Group N     │           │
│  │- 音乐音量    │ │- 导航音量   │ │- 电话音量   │           │
│  │- 增益控制    │ │- 静音管理   │ │- 渐变控制   │           │
│  └────┬────────┘ └───┬────────┘ └───┬────────┘           │
│       │              │              │                     │
│  ┌────▼──────────────▼──────────────▼─────────────────┐   │
│  │              CarAudioFocus (音频焦点)                │   │
│  │  - 焦点请求处理                                      │   │
│  │  - 焦点冲突解决                                      │   │
│  │  - 延迟焦点支持                                      │   │
│  │  - 焦点栈管理                                        │   │
│  └────────────────────────────────────────────────────┘   │
└────────────────────────┬───────────────────────────────────┘
                         │
┌────────────────────────┼───────────────────────────────────┐
│         配置层 (Configuration Layer)                        │
│                         │                                    │
│  ┌──────────────────────▼──────────────────────────────┐   │
│  │           CarAudioParameters (参数配置)               │   │
│  │  - 音频总线配置 (Bus Configuration)                  │   │
│  │  - 设备地址映射 (Device Address Mapping)             │   │
│  │  - 通道配置 (Channel Configuration)                  │   │
│  │  - HFP配置 (Hands-Free Profile)                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                         │                                    │
│  ┌──────────────────────▼──────────────────────────────┐   │
│  │           CarAudioSettings (设置持久化)               │   │
│  │  - 用户音量设置                                      │   │
│  │  - 静音状态保存                                      │   │
│  │  - 跨重启保持                                        │   │
│  └─────────────────────────────────────────────────────┘   │
│                         │                                    │
│  ┌──────────────────────▼──────────────────────────────┐   │
│  │        CarAudioContext / CarAudioUsages              │   │
│  │  - 音频上下文定义                                     │   │
│  │  - 使用场景映射                                       │   │
│  └─────────────────────────────────────────────────────┘   │
└────────────────────────┬───────────────────────────────────┘
                         │
┌────────────────────────┼───────────────────────────────────┐
│          HAL封装层 (HAL Wrapper Layer)                      │
│                         │                                    │
│  ┌──────────────────────▼──────────────────────────────┐   │
│  │         AudioControlWrapper (接口定义)                │   │
│  │  - AudioControlCommon (公共接口)                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                         │                                    │
│  ┌──────────────────────▼──────────────────────────────┐   │
│  │         AudioControlHal (抽象基类)                    │   │
│  └────┬──────────────────────┬──────────────────────┘   │
│       │                      │                           │
│  ┌────▼──────────────┐  ┌───▼────────────────┐          │
│  │ AudioControlBase  │  │AudioControlA2b     │          │
│  │ - 基础音频控制     │  │ - A2B总线控制       │          │
│  │ - 音量设置         │  │ - RPR配置加载       │          │
│  │ - 设备路由         │  │ - 声道控制          │          │
│  └───────────────────┘  └────────────────────┘          │
│                                                          │
│  ┌──────────────────────┐  ┌────────────────────┐       │
│  │AudioControlLocal     │  │HalAudioFocus       │       │
│  │ - 本地音频控制        │  │ - HAL层焦点管理     │       │
│  │ - 设备发现            │  │ - 焦点通知          │       │
│  └──────────────────────┘  └────────────────────┘       │
└────────────────────────┬─────────────────────────────────┘
                         │
┌────────────────────────┼─────────────────────────────────┐
│           HAL层 (Hardware Abstraction Layer)              │
│                         │                                  │
│  ┌──────────────────────▼────────────────────────────┐    │
│  │  Audio HAL (vendor/hardware接口)                    │    │
│  │  - IAudioControl HAL接口                           │    │
│  │  - 设备驱动交互                                     │    │
│  │  - 硬件参数设置                                     │    │
│  └───────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────┘
```

## 类关系图

### 服务类继承关系

```
CarServiceBase (接口)
    ▲
    │ implements
    │
┌───┴──────────┐  ┌───────────────┐  ┌──────────────┐
│CarAudioService│  │  A2bService   │  │ AscService   │
│              │  │               │  │              │
│ICarAudio.Stub│  │IA2b.Stub      │  │IAsc.Stub     │
└──────────────┘  └───────────────┘  └──────────────┘
```

### 音频管理层关系

```
CarAudioService
    │ contains
    ├──► CarAudioZone (1:N)
    │       │ contains
    │       ├──► CarVolumeGroup (1:N)
    │       │       │ uses
    │       │       ├──► CarAudioDeviceInfo
    │       │       └──► CarAudioSettings
    │       │
    │       └──► CarAudioFocus
    │               │ uses
    │               ├──► FocusEntry
    │               └──► FocusInteraction
    │
    └──► CarAudioParameters
            │ defines
            ├──► Bus
            ├──► BusAddress
            └──► Channel
```

### HAL层封装关系

```
AudioControlCommon (接口)
    ▲
    │ implements
    │
AudioControlHal (抽象类)
    ▲
    │ extends
    │
    ├──► AudioControlBase
    │       │ implements
    │       └──► AudioControlWrapper
    │
    ├──► AudioControlA2b
    │       │ implements
    │       └──► IAudioControlA2b
    │
    └──► AudioControlLocal
```

## 数据流详细图

### 音量控制流程

```
用户操作/App调用
    │
    ▼
┌─────────────────┐
│ CarAudioService │
│ setGroupVolume()│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  CarAudioZone   │
│ getVolumeGroup()│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ CarVolumeGroup  │
│ setVolume()     │
└────────┬────────┘
         │
         ├─► 计算增益索引
         │  gainIndex = volumeToGainIndex(volume)
         │
         ├─► 检查静音状态
         │  if (mMute) return;
         │
         └─► 设置HAL
            │
            ▼
    ┌──────────────┐
    │AudioControl  │
    │setVolume()   │
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │  Audio HAL   │
    │ (硬件驱动)    │
    └──────────────┘
           │
           ▼
    ┌──────────────┐
    │ 音量变化通知  │
    │ Callback     │
    └──────────────┘
```

### 音频焦点流程

```
App请求焦点
requestAudioFocus()
    │
    ▼
┌──────────────────┐
│ CarAudioService  │
│ requestAudioFocus│
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  CarAudioZone    │
│  获取对应Zone     │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  CarAudioFocus   │
│  requestFocus()  │
└────────┬─────────┘
         │
         ├─► 检查焦点策略
         │  isFocusAllowed()?
         │
         ├─► 通知当前持有者
         │  sendFocusLoss(LOSS_TRANSIENT)
         │
         ├─► 更新焦点栈
         │  mFocusHolders.put(clientId, entry)
         │
         └─► 授予新焦点
            │
            ▼
    ┌──────────────┐
    │sendFocusGain │
    │(GAIN_FOCUS)  │
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │ App接收回调   │
    │ onAudioFocus │
    └──────────────┘
```

### A2B转向提示音流程

```
导航系统
发送转向信息
    │
    ▼
┌──────────────────┐
│ IMapTbtManager   │
│ NaviTbtListener  │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│   A2bService     │
│ onTurnInfo()     │
└────────┬─────────┘
         │
         ├─► 查找转向映射表
         │  TURNKINDID_MAP[turnKind]
         │
         ├─► 解析声道配置
         │  {左, 右, 中, 类型ID}
         │  {ACTIVE, OFF, OFF, 7} // 左转
         │
         └─► 应用配置
            │
            ▼
    ┌──────────────────┐
    │AudioControlA2b   │
    │setChannelConfig()│
    └────────┬─────────┘
             │
             ▼
    ┌──────────────────┐
    │  加载RPR配置      │
    │  amp.rpr         │
    │  anc.rpr         │
    └────────┬─────────┘
             │
             ▼
    ┌──────────────────┐
    │ 播放转向提示音    │
    │ 空间音频输出      │
    └──────────────────┘
```

## 配置文件结构

```
car_audio_configuration.xml
├── <audioZone id="0" name="主驾驶区">
│   ├── <volumeGroup id="0">
│   │   ├── <context>MUSIC</context>
│   │   ├── <context>NAVIGATION</context>
│   │   └── <device>bus00</device>
│   │
│   └── <volumeGroup id="1">
│       ├── <context>CALL</context>
│       └── <device>bus01</device>
│
├── <audioZone id="1" name="副驾驶区">
│   └── <volumeGroup id="2">
│       ├── <context>MUSIC</context>
│       └── <device>bus02</device>
│
└── <audioZone id="2" name="后排区">
    └── <volumeGroup id="3">
        ├── <context>MUSIC</context>
        └── <device>bus03</device>
```

## 关键接口定义

### ICarAudio (Binder接口)

```java
interface ICarAudio {
    // 音量控制
    int getGroupVolume(int zoneId, int groupId);
    void setGroupVolume(int zoneId, int groupId, int volume);
    int getGroupVolumeMax(int zoneId, int groupId);
    
    // 焦点管理
    int requestAudioFocus(AudioAttributes attr, int durationHint);
    void abandonAudioFocus(AudioAttributes attr);
    
    // 设备管理
    List<AudioDeviceInfo> getAudioDeviceInfos(int zoneId);
    
    // 静音控制
    boolean isGroupMuted(int zoneId, int groupId);
    void setGroupMute(int zoneId, int groupId, boolean mute);
}
```

### IA2b (A2B服务接口)

```java
interface IA2b {
    // 转向提示
    void onTurnInfo(int turnKind);
    
    // RPR配置
    void loadRprFile(String path);
    
    // 声道控制
    void setChannelConfig(int left, int right, int center);
    
    // 回调注册
    void registerCallback(IUpdateCallback callback);
}
```

### IAsc (ASC服务接口)

```java
interface IAsc {
    // 使能控制
    boolean setEnable(boolean enable);
    boolean isEnable();
    
    // 参数配置
    void setParameters(String params);
}
```

## 文件依赖关系图

```
CarAudioService.java
├── CarAudioZone.java
│   ├── CarVolumeGroup.java
│   │   ├── CarAudioDeviceInfo.java
│   │   └── CarAudioSettings.java
│   └── CarAudioFocus.java
│       ├── FocusEntry.java
│       └── FocusInteraction.java
│
├── CarAudioParameters.java
│   ├── CarAudioContext.java
│   └── CarAudioUsages.java
│
├── hal/
│   ├── AudioControlBase.java
│   ├── AudioControlA2b.java
│   ├── AudioControlLocal.java
│   └── HalAudioFocus.java
│
├── A2bService.java
│   ├── A2bCallbackHandler.java
│   └── RprFile.java
│
├── AscService.java
│   └── AscHalService.java
│
└── 辅助类
    ├── CarMute.java
    ├── CarVolume.java
    ├── SoundPlayer.java
    └── *CallbackHandler.java
```

## 总结

Car Audio 模块采用分层架构设计：
- **服务层**: 提供Binder接口，处理跨进程调用
- **管理层**: 实现核心业务逻辑（焦点、音量、区域）
- **配置层**: 管理参数和持久化设置
- **HAL层**: 封装硬件抽象层接口

这种设计实现了：
1. 关注点分离，职责清晰
2. 良好的可扩展性
3. 配置驱动的灵活性
4. HAL层的可替换性

--------
# Car Audio 服务代码梳理

## 目录位置
`/home/.../android12/packages/services/Car/service/src/com/android/car/audio/`

## 一、整体架构概述

Car Audio 模块是 Android Car 系统中负责音频管理的核心服务，主要功能包括：
- 音频焦点管理（Audio Focus）
- 音量控制（Volume Control）
- 音频区域管理（Audio Zones）
- HAL层音频控制
- A2B/ASC 特殊音频服务

## 二、核心服务类

### 2.1 CarAudioService（主服务）
**文件**: `CarAudioService.java` (2907行, 117.7KB)

**职责**: 
- Car音频系统的主服务类
- 实现 `ICarAudio.Stub` Binder接口
- 实现 `CarServiceBase` 服务生命周期接口

**核心功能**:
- 音频配置加载（从 `/vendor/etc/car_audio_configuration.xml`）
- 音频区域（Audio Zone）管理
- 音频焦点策略控制
- 音量组（Volume Group）管理
- 音频路由和混音
- 与HAL层交互

**关键成员**:
```java
- mContext: Context
- mAudioManager: AudioManager
- mCarAudioParameters: 音频参数配置
- mCarAudioZones: 音频区域列表
- mCarAudioFocus: 音频焦点管理
- mAudioControl: HAL层音频控制
```

### 2.2 A2bService（A2B音频服务）
**文件**: `A2bService.java` (651行, 26.3KB)

**职责**:
- A2B（Automotive Audio Bus）音频总线服务
- 实现 `IA2b.Stub` 接口
- 处理导航转向提示音的空间音频分发

**核心功能**:
- 加载AMP和ANC的RPR配置文件
- 根据导航转向类型控制左右声道输出
- 支持42种转向类型的音频映射
- 与MapTbt（导航转向）服务联动

**转向类型映射示例**:
```java
{ ACTIVE,ACTIVE,OFF,  1 }, // 直行 - 左右都激活
{ OFF,ACTIVE,OFF,     3 }, // 右转 - 只激活右声道
{ ACTIVE,OFF,OFF,     7 }, // 左转 - 只激活左声道
```

### 2.3 AscService（ASC音频服务）
**文件**: `AscService.java` (150行, 4.8KB)

**职责**:
- ASC（Active Sound Control）主动声音控制服务
- 实现 `IAsc.Stub` 接口
- 控制发动机声音增强

**核心功能**:
- 启动/停止ASC HAL服务
- 通过系统属性持久化使能状态
- 根据电源状态自动恢复
- 动态配置音频参数

## 三、音频管理核心类

### 3.1 CarAudioFocus（音频焦点）
**文件**: `CarAudioFocus.java` (753行, 35.2KB)

**职责**: 管理车内音频焦点策略

**核心功能**:
- 音频焦点请求和释放
- 焦点冲突处理
- 延迟焦点请求支持
- 焦点持有者追踪
- 与AudioManager交互

**焦点管理**:
```java
- mFocusHolders: 当前持有焦点的应用
- mFocusLosers: 失去焦点的应用
- mAudioPolicy: 音频策略
- mFocusInteraction: 焦点交互接口
```

### 3.2 CarAudioZone（音频区域）
**文件**: `CarAudioZone.java` (363行, 12.4KB)

**职责**: 封装车内音频区域概念

**核心功能**:
- 管理多个音量组（Volume Group）
- 独立的音频焦点实例
- 音频设备信息管理
- 主/次区域区分

**结构**:
```
CarAudioZone (区域)
  ├── CarVolumeGroup 1 (音量组1)
  ├── CarVolumeGroup 2 (音量组2)
  └── CarAudioFocus (焦点管理)
```

### 3.3 CarVolumeGroup（音量组）
**文件**: `CarVolumeGroup.java` (625行, 21.5KB)

**职责**: 管理音量组的音量控制

**核心功能**:
- 音量增益控制
- 静音管理
- 音频上下文映射
- 设备地址管理
- 音量渐变（Fade In）

**音量控制**:
```java
- mCurrentGainIndex: 当前增益索引
- mMaxGainIndex: 最大增益
- mMinGainIndex: 最小增益
- mMute: 静音状态
- mContextToAddress: 上下文到设备地址映射
```

## 四、HAL层封装

### 4.1 HAL目录结构
```
hal/
├── AudioControlBase.java      - 基础音频控制实现
├── AudioControlHal.java       - HAL抽象基类
├── AudioControlA2b.java       - A2B音频控制
├── AudioControlLocal.java     - 本地音频控制
├── AudioControlFactory.java   - 工厂类
├── AudioControlWrapper.java   - 包装接口V1
├── AudioControlWrapperV2.java - 包装接口V2
├── HalAudioFocus.java         - HAL音频焦点
├── IAudioControlA2b.java      - A2B控制接口
└── AscHalService.java         - ASC HAL服务
```

### 4.2 继承关系
```
AudioControlCommon (接口)
  └── AudioControlHal (抽象类)
       ├── AudioControlBase (基础实现)
       │    └── AudioControlWrapper (实现)
       ├── AudioControlA2b (A2B控制)
       └── AudioControlLocal (本地控制)
```

## 五、配置和参数类

### 5.1 CarAudioParameters
**文件**: `CarAudioParameters.java` (76.8KB)

**职责**: 音频系统参数配置

**核心内容**:
- 音频总线（Bus）配置
- 设备地址映射
- 通道配置
- 音频上下文定义
- HFP（免提）配置

### 5.2 CarAudioSettings
**文件**: `CarAudioSettings.java` (9.8KB)

**职责**: 音频设置持久化管理

**功能**:
- 用户音量设置保存
- 静音状态存储
- 跨重启保持配置

### 5.3 CarAudioContext
**文件**: `CarAudioContext.java` (17.4KB)

**职责**: 定义音频上下文类型

**常见上下文**:
- `CONTEXT_MUSIC` - 音乐
- `CONTEXT_NAVIGATION` - 导航
- `CONTEXT_VOICE_COMMAND` - 语音命令
- `CONTEXT_CALL` - 电话
- `CONTEXT_ALARM` - 警报

### 5.4 CarAudioUsages
**文件**: `CarAudioUsages.java` (54.0KB)

**职责**: 音频使用场景映射

**功能**:
- AudioAttributes.Usage 到 CarAudioContext 的映射
- 系统使用场景定义
- 自定义使用场景

## 六、辅助处理类

### 6.1 回调处理器
- **A2bCallbackHandler.java** (4.9KB) - A2B回调处理
- **CarVolumeCallbackHandler.java** (5.5KB) - 音量回调处理
- **CarAudioFocusListenerHandler.java** (3.7KB) - 焦点回调处理
- **CarSoundCallbackHandler.java** (6.0KB) - 声音回调处理

### 6.2 其他辅助类
- **CarMute.java** (5.9KB) - 静音管理
- **CarVolume.java** (7.9KB) - 音量工具类
- **CarUsageParams.java** (5.2KB) - 使用参数
- **FocusEntry.java** (4.7KB) - 焦点条目
- **FocusInteraction.java** (13.2KB) - 焦点交互
- **SoundPlayer.java** (3.1KB) - 声音播放
- **CarAudioDeviceInfo.java** (8.4KB) - 设备信息

### 6.3 区域辅助类
- **CarAudioZonesHelper.java** (26.7KB) - 区域辅助
- **CarAudioZonesHelperLegacy.java** (9.1KB) - 旧版区域辅助
- **CarAudioZonesValidator.java** (2.4KB) - 区域验证器
- **CarAudioDynamicRouting.java** (5.7KB) - 动态路由
- **CarAudioPatchHandle.java** - 音频补丁句柄

## 七、数据流向

```
应用层 App
    ↓ (Binder IPC)
CarAudioService (ICarAudio.Stub)
    ↓
├── CarAudioZone (区域管理)
│   ├── CarVolumeGroup (音量控制)
│   └── CarAudioFocus (焦点管理)
    ↓
├── CarAudioParameters (参数配置)
├── CarAudioSettings (设置持久化)
└── AudioControlWrapper (HAL接口)
    ↓
├── AudioControlBase (基础控制)
├── AudioControlA2b (A2B控制)
└── AudioControlLocal (本地控制)
    ↓
HAL层 (vendor/hardware)
```

## 八、关键流程

### 8.1 服务初始化流程
1. CarAudioService 创建
2. 加载音频配置文件（car_audio_configuration.xml）
3. 初始化 CarAudioParameters
4. 创建 CarAudioZone 列表
5. 为每个Zone创建 CarVolumeGroup
6. 初始化 CarAudioFocus
7. 连接HAL层 AudioControl

### 8.2 音频焦点请求流程
1. App 请求音频焦点
2. CarAudioService 接收请求
3. 转发到对应Zone的 CarAudioFocus
4. CarAudioFocus 检查焦点策略
5. 通知当前焦点持有者（LOSS）
6. 授予新请求者焦点（GAIN）
7. 更新焦点栈

### 8.3 音量调整流程
1. App 或用户调整音量
2. CarAudioService 接收请求
3. 查找对应的 CarVolumeGroup
4. CarVolumeGroup 计算增益索引
5. 通过 AudioControl 设置HAL层音量
6. 通知音量变化回调

### 8.4 A2B转向提示音流程
1. 导航系统发送转向信息
2. A2bService 接收转向类型
3. 查找转向类型映射表
4. 配置左右声道激活状态
5. 通过 AudioControlA2b 应用配置
6. 播放转向提示音

## 九、配置文件

### 9.1 音频配置路径（优先级）
1. `/vendor/etc/car_audio_configuration.xml`
2. `/system/etc/car_audio_configuration.xml`
3. 回退到资源文件 `car_volume_groups.xml`

### 9.2 配置内容
- 音频区域定义
- 音量组配置
- 音频上下文映射
- 设备地址分配
- 增益范围设置

## 十、定制功能

### 10.1 3QK项目特定代码
```java
private static final boolean IS_3QK = "3qk".equals(
    SystemProperties.get("ro.product.code", null));
```

### 10.2 A2B导航联动
- 与 IMapTbtManager 集成
- 根据导航转向类型控制音频输出
- 支持空间音频提示

### 10.3 ASC发动机声音增强
- 通过系统属性控制使能
- 动态音频参数配置
- 电源状态联动

## 十一、统计信息

| 类别 | 文件数 | 总行数 | 总大小 |
|------|--------|--------|--------|
| 核心服务 | 3 | ~4,300 | ~150KB |
| 音频管理 | 8 | ~3,500 | ~180KB |
| HAL封装 | 12 | ~1,500 | ~120KB |
| 配置参数 | 4 | ~2,000 | ~160KB |
| 辅助类 | 12 | ~1,500 | ~60KB |
| **总计** | **39** | **~12,800** | **~670KB** |

## 十二、关键技术点

1. **音频焦点策略**: 自定义焦点管理，支持延迟焦点请求
2. **多区域支持**: 支持车内多个独立音频区域
3. **音量分组**: 按上下文分组控制音量
4. **HAL抽象**: 多层HAL封装，支持不同版本
5. **空间音频**: A2B总线支持转向提示音空间分发
6. **配置驱动**: XML配置驱动，灵活适配不同车型
7. **电源管理**: 与CarPowerManager联动，支持休眠唤醒

## 十三、依赖关系

### 13.1 Android系统依赖
- `android.media.AudioManager`
- `android.media.AudioSystem`
- `android.media.audiopolicy.AudioPolicy`
- `android.car.media.CarAudioManager`

### 13.2 Car服务依赖
- `CarServiceBase` - 服务基类
- `CarPowerManager` - 电源管理
- `CarOccupantZoneService` - 乘员区域服务

### 13.3 company定制依赖
- `Mgr` - 服务管理器
- `IA2b` - A2B接口
- `IAsc` - ASC接口
- `IMapTbtManager` - 导航转向管理

## 十四、扩展建议

1. **日志优化**: 添加更详细的调试日志
2. **性能监控**: 添加音频延迟统计
3. **异常处理**: 增强HAL断开恢复机制
4. **测试覆盖**: 添加单元测试和集成测试
5. **文档完善**: 补充API文档和配置说明
