# Android Automotive 音频各层对应关系详解

> 基于 Qualcomm SDM855/MSMnile Auto 平台代码分析，梳理从 CarService Audio Bus → HAL Usecase → ADSP → 驱动 PCM 节点的完整对应链路。

---

## 1. CarService 层（CarAudioConfiguration）

配置文件：`vendor/qcom/opensource/audio-hal-ar/primary-hal/configs/common_au/car_audio_configuration.xml`

CarAudioService 通过 `car_audio_configuration.xml` 将 Audio Context 映射到 Bus Address：

| Audio Context | Bus Address | Zone |
|---|---|---|
| music | BUS00_MEDIA | Primary Zone |
| alarm, notification, system_sound, emergency, safety, vehicle_status, announcement | BUS01_SYS_NOTIFICATION | Primary Zone |
| navigation, voice_command | BUS02_NAV_GUIDANCE | Primary Zone |
| call, call_ring | BUS03_PHONE | Primary Zone |
| 全部 context | BUS08_FRONT_PASSENGER | Front Passenger Zone |
| 全部 context | BUS16_REAR_SEAT | Rear Seat Zone |

配置示例：

```xml
<zone name="primary zone" isPrimary="true" occupantZoneId="0">
    <volumeGroups>
        <group>
            <device address="BUS00_MEDIA">
                <context context="music"/>
            </device>
        </group>
        <group>
            <device address="BUS01_SYS_NOTIFICATION">
                <context context="alarm"/>
                <context context="notification"/>
                <context context="system_sound"/>
            </device>
        </group>
        <group>
            <device address="BUS02_NAV_GUIDANCE">
                <context context="navigation"/>
                <context context="voice_command"/>
            </device>
        </group>
        <group>
            <device address="BUS03_PHONE">
                <context context="call"/>
                <context context="call_ring"/>
            </device>
        </group>
    </volumeGroups>
</zone>
```

---

## 2. Audio Policy 层（audio_policy_configuration.xml）

配置文件：`vendor/qcom/opensource/audio-hal-ar/primary-hal/configs/common_au/audio_policy_configuration.xml`

**devicePort**（类型 `AUDIO_DEVICE_OUT_BUS`）通过 `address` 属性关联到 CarService 的 Bus Address，**mixPort** 定义输出流，**route** 将两者连接。

### devicePort 定义

| devicePort tagName | type | address |
|---|---|---|
| Media Bus | AUDIO_DEVICE_OUT_BUS | BUS00_MEDIA |
| Sys Notification Bus | AUDIO_DEVICE_OUT_BUS | BUS01_SYS_NOTIFICATION |
| Nav Guidance Bus | AUDIO_DEVICE_OUT_BUS | BUS02_NAV_GUIDANCE |
| Phone Bus | AUDIO_DEVICE_OUT_BUS | BUS03_PHONE |
| Front Passenger Bus | AUDIO_DEVICE_OUT_BUS | BUS08_FRONT_PASSENGER |
| Rear Seat Bus | AUDIO_DEVICE_OUT_BUS | BUS16_REAR_SEAT |

### mixPort 定义

| mixPort name | flags | 采样率 | 通道 |
|---|---|---|---|
| media | AUDIO_OUTPUT_FLAG_PRIMARY | 48000 | STEREO |
| sys_notification | 无 | 48000 | STEREO |
| nav_guidance | 无 | 48000 | STEREO |
| phone | 无 | 48000 | STEREO |
| front_passenger | 无 | 48000 | STEREO |
| rear_seat | 无 | 48000 | STEREO |

### Route 连接（mixPort → devicePort）

```xml
<route type="mix" sink="Media Bus"           sources="media"/>
<route type="mix" sink="Sys Notification Bus" sources="sys_notification"/>
<route type="mix" sink="Nav Guidance Bus"     sources="nav_guidance"/>
<route type="mix" sink="Phone Bus"            sources="phone"/>
<route type="mix" sink="Front Passenger Bus"  sources="front_passenger"/>
<route type="mix" sink="Rear Seat Bus"        sources="rear_seat"/>
```

---

## 3. Audio HAL 层（Usecase + snd_device + PCM Device）

### 3.1 Bus Address → car_audio_stream

源文件：`vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/audio_extn/auto_hal.c`

`auto_hal_get_car_audio_stream_from_address()` 函数从 address 提取 bus 号，映射为 `car_audio_stream`：

```c
int auto_hal_get_car_audio_stream_from_address(const char *address)
{
    int bus_num = -1;
    char *str = NULL;
    char *last_r = NULL;
    char local_address[AUDIO_DEVICE_MAX_ADDRESS_LEN];

    strlcpy(local_address, address, AUDIO_DEVICE_MAX_ADDRESS_LEN);

    /* extract bus number from address */
    str = strtok_r(local_address, "BUS_", &last_r);
    if (str != NULL)
        bus_num = (int)strtol(str, (char **)NULL, 10);

    return (0x1 << bus_num);  // 返回 bit flag
}
```

| Bus Address | bus_num | car_audio_stream (bit flag) |
|---|---|---|
| BUS00_MEDIA | 0 | CAR_AUDIO_STREAM_MEDIA = 0x1 |
| BUS01_SYS_NOTIFICATION | 1 | CAR_AUDIO_STREAM_SYS_NOTIFICATION = 0x2 |
| BUS02_NAV_GUIDANCE | 2 | CAR_AUDIO_STREAM_NAV_GUIDANCE = 0x4 |
| BUS03_PHONE | 3 | CAR_AUDIO_STREAM_PHONE = 0x8 |
| BUS08_FRONT_PASSENGER | 8 | CAR_AUDIO_STREAM_FRONT_PASSENGER = 0x100 |
| BUS16_REAR_SEAT | 16 | CAR_AUDIO_STREAM_REAR_SEAT = 0x10000 |

### 3.2 car_audio_stream → Usecase → PCM Device ID

源文件：`auto_hal.c` → `auto_hal_open_output_stream()` 和 `platform.c` → `pcm_device_table[]`

| car_audio_stream | usecase | PCM Device ID | 说明 |
|---|---|---|---|
| CAR_AUDIO_STREAM_MEDIA | USECASE_AUDIO_PLAYBACK_MEDIA | 0 (DEEP_BUFFER_PCM_DEVICE) | 共享 deep-buffer |
| CAR_AUDIO_STREAM_SYS_NOTIFICATION | USECASE_AUDIO_PLAYBACK_SYS_NOTIFICATION | 9 | 独立低延迟通道 |
| CAR_AUDIO_STREAM_NAV_GUIDANCE | USECASE_AUDIO_PLAYBACK_NAV_GUIDANCE | 1 (MULTIMEDIA2_PCM_DEVICE) | 多媒体通道2 |
| CAR_AUDIO_STREAM_PHONE | USECASE_AUDIO_PLAYBACK_PHONE | 12 (LOWLATENCY_PCM_DEVICE) | 低延迟通道 |
| CAR_AUDIO_STREAM_FRONT_PASSENGER | USECASE_AUDIO_PLAYBACK_FRONT_PASSENGER | 55 | 独立PCM |
| CAR_AUDIO_STREAM_REAR_SEAT | USECASE_AUDIO_PLAYBACK_REAR_SEAT | 54 | 独立PCM |

PCM Device ID 就是 ALSA 驱动层的 `/dev/snd/pcmCxDxp` 节点号：

- PCM Device 0 → `/dev/snd/pcmC0D0p` (Media)
- PCM Device 1 → `/dev/snd/pcmC0D1p` (Nav Guidance)
- PCM Device 9 → `/dev/snd/pcmC0D9p` (Sys Notification)
- PCM Device 12 → `/dev/snd/pcmC0D12p` (Phone)
- PCM Device 54 → `/dev/snd/pcmC0D54p` (Rear Seat)
- PCM Device 55 → `/dev/snd/pcmC0D55p` (Front Passenger)

`pcm_device_table[]` 中的映射代码：

```c
static int pcm_device_table[AUDIO_USECASE_MAX][2] = {
    ...
    [USECASE_AUDIO_PLAYBACK_MEDIA] = {MEDIA_PCM_DEVICE, MEDIA_PCM_DEVICE},
    [USECASE_AUDIO_PLAYBACK_SYS_NOTIFICATION] = {SYS_NOTIFICATION_PCM_DEVICE, SYS_NOTIFICATION_PCM_DEVICE},
    [USECASE_AUDIO_PLAYBACK_NAV_GUIDANCE] = {NAV_GUIDANCE_PCM_DEVICE, NAV_GUIDANCE_PCM_DEVICE},
    [USECASE_AUDIO_PLAYBACK_PHONE] = {PHONE_PCM_DEVICE, PHONE_PCM_DEVICE},
    [USECASE_AUDIO_PLAYBACK_FRONT_PASSENGER] = {FRONT_PASSENGER_PCM_DEVICE, FRONT_PASSENGER_PCM_DEVICE},
    [USECASE_AUDIO_PLAYBACK_REAR_SEAT] = {REAR_SEAT_PCM_DEVICE, REAR_SEAT_PCM_DEVICE},
    ...
};
```

### 3.3 car_audio_stream → snd_device → ACDB ID

源文件：`auto_hal.c` → `auto_hal_get_snd_device_for_car_audio_stream()` 和 `auto_hal_get_output_snd_device()`

| car_audio_stream | snd_device | ACDB ID | mixer device name |
|---|---|---|---|
| CAR_AUDIO_STREAM_MEDIA | SND_DEVICE_OUT_BUS_MEDIA | 60 | "bus-speaker" |
| CAR_AUDIO_STREAM_SYS_NOTIFICATION | SND_DEVICE_OUT_BUS_SYS | 60 | "bus-speaker" |
| CAR_AUDIO_STREAM_NAV_GUIDANCE | SND_DEVICE_OUT_BUS_NAV | 14 | "bus-speaker" |
| CAR_AUDIO_STREAM_PHONE | SND_DEVICE_OUT_BUS_PHN | 94 | "bus-speaker" |
| CAR_AUDIO_STREAM_FRONT_PASSENGER | SND_DEVICE_OUT_BUS_PAX | - | "bus-speaker" |
| CAR_AUDIO_STREAM_REAR_SEAT | SND_DEVICE_OUT_BUS_RSE | 60 | "bus-speaker" |

ACDB ID 用于在 ADSP 端选择对应的音频处理拓扑（topology）和校准数据。

---

## 4. Mixer Paths 层（mixer_paths_adp.xml）

配置文件：`vendor/qcom/opensource/audio-hal-ar/primary-hal/configs/msmnile_au/mixer_paths_adp.xml`

对于 Auto 平台，默认输出设备 "speaker" 实际对应 `TERT_TDM_RX_0`（Tertiary TDM），mixer path 通过 **FE PCM → BE AIF** 的方式连接。

### FE (Front-End) → BE (Back-End) 对应关系

| path name | FE (MultiMedia) | FE PCM ID | BE (AIF) | 说明 |
|---|---|---|---|---|
| deep-buffer-playback | MultiMedia1 | 0 | TERT_TDM_RX_0 | Media Bus 默认输出 |
| multi-channel-playback | MultiMedia2 | 1 | TERT_TDM_RX_0 | Nav Guidance |
| low-latency-playback | MultiMedia5 | 9 | TERT_TDM_RX_0 | Sys Notification / Phone |
| compress-offload-playback | MultiMedia4 | - | TERT_TDM_RX_0 | Offload 输出 |
| audio-ull-playback | MultiMedia8 | - | TERT_TDM_RX_0 | ULL 输出 |

### Mixer Control 示例

```xml
<!-- deep-buffer-playback: FE MultiMedia1 → BE TERT_TDM_RX_0 -->
<path name="deep-buffer-playback">
    <ctl name="TERT_TDM_RX_0 Channels" value="Six"/>
    <ctl name="TERT_TDM_RX_0 Audio Mixer MultiMedia1" value="1"/>
</path>

<!-- low-latency-playback: FE MultiMedia5 → BE TERT_TDM_RX_0 -->
<path name="low-latency-playback">
    <ctl name="TERT_TDM_RX_0 Channels" value="Six"/>
    <ctl name="TERT_TDM_RX_0 Audio Mixer MultiMedia5" value="1"/>
</path>

<!-- audio-ull-playback: FE MultiMedia8 → BE TERT_TDM_RX_0 -->
<path name="audio-ull-playback">
    <ctl name="TERT_TDM_RX_0 Channels" value="Six"/>
    <ctl name="TERT_TDM_RX_0 Audio Mixer MultiMedia8" value="1"/>
</path>
```

### 其他 BE（Back-End）AIF

| BE AIF | 用途 | 对应场景 |
|---|---|---|
| TERT_TDM_RX_0 | 主输出（Speaker） | 所有 Bus 的默认输出 |
| SLIMBUS_6_RX | 耳机输出 | headphones |
| SLIMBUS_7_RX | BT SCO 输出 | bt-sco |
| DISPLAY_PORT | DP 输出 | display-port |
| USB_AUDIO_RX | USB 输出 | usb-headphones |
| AFE_PCM_RX | AFE Proxy 输出 | afe-proxy |

---

## 5. ADSP 层

ADSP (Audio DSP) 固件位于 `adsp_8155/adsp_proc/` 目录，通过 APR/GPR 通道与 HAL 通信。

ADSP 侧通过 ACDB ID 选择对应的 topology 和音频处理模块：

| ACDB ID | 用途 | 说明 |
|---|---|---|
| 60 | Media / Sys Notification / Rear Seat | 默认处理拓扑（标准播放） |
| 14 | Navigation | 专用拓扑（可能带有混音/ducking 处理） |
| 94 | Phone | 专用拓扑（可能带有 EC/NS 回声消除） |

ADSP 内部关键概念：

- **Topology**：定义了音频信号的处理流程（解码 → DSP处理 → 混音 → 输出路由）
- **ACDB (Audio Calibration Database)**：存储各 snd_device 的校准参数
- **APR/GPR**：应用处理器与 ADSP 之间的通信协议
- **ASM (Audio Stream Manager)**：管理音频流
- **ADM (Audio Device Manager)**：管理音频设备路由

---

## 6. 完整对应链路图

```
┌─────────────────────────────────────────────────────────────────┐
│  CarService (car_audio_configuration.xml)                       │
│  Context → Bus Address                                          │
│  music → BUS00_MEDIA                                            │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  AudioPolicy (audio_policy_configuration.xml)                   │
│  devicePort address → mixPort name                              │
│  BUS00_MEDIA (AUDIO_DEVICE_OUT_BUS) → "media"                   │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  Audio HAL (auto_hal.c / platform.c)                            │
│  Address → car_audio_stream → usecase → PCM Dev → snd_device   │
│  BUS00_MEDIA → STREAM_MEDIA → PLAYBACK_MEDIA → PCM0 → BUS_MEDIA│
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  ALSA Driver                                                    │
│  /dev/snd/pcmC0D0p (FE PCM device 0 = MultiMedia1)             │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  Mixer Control (mixer_paths_adp.xml)                            │
│  FE MultiMedia1 → TERT_TDM_RX_0 Audio Mixer                    │
│  (将 FE PCM 流路由到 BE TDM 输出)                                │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  ADSP (ACDB ID 60)                                              │
│  选择 topology → 音频处理 → 输出到 TDM                           │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  Hardware (Amplifier → Speaker)                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7. 核心对应关系总表

| Bus Address | Context | mixPort | usecase | PCM Dev | snd_device | ACDB ID | FE→BE Mixer |
|---|---|---|---|---|---|---|---|
| BUS00_MEDIA | music | media | PLAYBACK_MEDIA | 0 | OUT_BUS_MEDIA | 60 | MM1→TERT_TDM_RX_0 |
| BUS01_SYS_NOTIFICATION | alarm/notification/system_sound | sys_notification | PLAYBACK_SYS_NOTIFICATION | 9 | OUT_BUS_SYS | 60 | MM5→TERT_TDM_RX_0 |
| BUS02_NAV_GUIDANCE | navigation/voice_command | nav_guidance | PLAYBACK_NAV_GUIDANCE | 1 | OUT_BUS_NAV | 14 | MM2→TERT_TDM_RX_0 |
| BUS03_PHONE | call/call_ring | phone | PLAYBACK_PHONE | 12 | OUT_BUS_PHN | 94 | MM5→TERT_TDM_RX_0 |
| BUS08_FRONT_PASSENGER | all | front_passenger | PLAYBACK_FRONT_PASSENGER | 55 | OUT_BUS_PAX | - | - |
| BUS16_REAR_SEAT | all | rear_seat | PLAYBACK_REAR_SEAT | 54 | OUT_BUS_RSE | 60 | - |

---

## 8. 关键源文件索引

| 文件 | 路径 | 作用 |
|---|---|---|
| car_audio_configuration.xml | `vendor/qcom/opensource/audio-hal-ar/primary-hal/configs/common_au/` | CarService Context→Bus映射 |
| audio_policy_configuration.xml | `vendor/qcom/opensource/audio-hal-ar/primary-hal/configs/common_au/` | devicePort/mixPort/route定义 |
| audio_policy_configuration.xml | `vendor/qcom/opensource/audio-hal-ar/primary-hal/configs/msmnile_au/` | msmnile平台特定策略配置 |
| auto_hal.c | `vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/audio_extn/` | Bus→usecase→snd_device映射 |
| auto_hal.h | `vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/audio_extn/` | Auto HAL接口定义 |
| audio_hw.h | `vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/` | usecase/snd_device枚举定义 |
| platform.h | `vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/msm8974/` | PCM Device ID宏定义 |
| platform.c | `vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/msm8974/` | pcm_device_table/acdb_id映射 |
| mixer_paths_adp.xml | `vendor/qcom/opensource/audio-hal-ar/primary-hal/configs/msmnile_au/` | FE→BE mixer路由控制 |
| mixer_paths.xml | `vendor/qcom/opensource/audio-hal-ar/primary-hal/configs/msmnile_au/` | 默认mixer路由控制 |

---

## 9. 补充说明

### 9.1 Bus Number 编码规则

Bus Address 格式为 `BUS<NUM>_<NAME>`，其中 `<NUM>` 的含义：

- **0-7**：Primary Zone（主驾区域）
- **8-15**：Front Passenger Zone（副驾区域）
- **16-23**：Rear Seat Zone（后排区域）

`car_audio_stream` 通过 `0x1 << bus_num` 生成 bit flag，这样不同 Zone 的 stream 不会冲突。

### 9.2 多 Bus 共享同一 BE 输出

在 msmnile_au 平台上，多个 Bus（Media/Nav/Phone/Sys Notification）共享同一个 BE 输出 `TERT_TDM_RX_0`，通过不同的 FE PCM（MultiMedia1/2/5/8）区分。ADSP 内部负责将多路 FE 流混音后输出到 TDM。

### 9.3 Volume 控制

- CarService 通过 AudioPort 设置 gain（millibel 单位）
- HAL 层将 millibel 转换为线性增益（`powf(10.0f, mdb/2000)`）
- 对于 BUS 设备，gain 应用到 DSP mixer；对于外部设备，gain 应用到 CODEC amplifier

---

## 10. 音频播放完整流程（应用层→Framework→Native→HAL→驱动→ADSP→硬件）

以下以 **Media Bus (BUS00_MEDIA)** 为例，追踪音频从应用到硬件的完整播放流程。

---

### 10.1 总体流程图

```
┌──────────────────────────────────────────────────────────────────────────┐
│  应用层 (AudioTrack)                                                     │
│  new AudioTrack() → write() → setVolume()                               │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │ Binder IPC
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  Framework层 (AudioPolicyService + AudioFlinger)                         │
│  AudioPolicyService::getOutput() → AudioFlinger::openOutput()           │
│  AudioFlinger::PlaybackThread::write()                                   │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │ dlopen + HIDL/AIDL
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  Audio HAL层 (audio_hw.c / auto_hal.c / platform.c)                     │
│  adev_open_output_stream() → start_output_stream() → select_devices()   │
│  → enable_snd_device() → enable_audio_route() → pcm_open() → pcm_write()│
└───────────────────────────────┬──────────────────────────────────────────┘
                                │ /dev/snd/pcmC0Dxp
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  ALSA 驱动层 (Kernel)                                                    │
│  snd_pcm_open() → snd_pcm_writei() → FE PCM (MultiMedia1)              │
│  → ALSA mixer → BE AIF (TERT_TDM_RX_0)                                 │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │ APR/GPR
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  ADSP层                                                                  │
│  ASM (Audio Stream Manager) → ADM (Audio Device Manager)                │
│  → ACDB calibration (ACDB ID 60) → topology → DSP处理 → TDM输出         │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │ I2S/TDM
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  硬件层 (Amplifier → Speaker)                                            │
└──────────────────────────────────────────────────────────────────────────┘
```

---

### 10.2 打开音频链路

#### 步骤1：应用层创建 AudioTrack

```java
// frameworks/base/media/java/android/media/AudioTrack.java
AudioTrack track = new AudioTrack(
    AudioManager.STREAM_MUSIC,      // stream type → CarService映射到 music context → BUS00_MEDIA
    48000,                          // sample rate
    AudioFormat.CHANNEL_OUT_STEREO, // channel config
    AudioFormat.ENCODING_PCM_16BIT, // format
    bufferSize,                     // buffer size
    AudioTrack.MODE_STREAM          // mode
);
track.play();                       // 开始播放
```

#### 步骤2：AudioPolicyService 路由选择

AudioPolicyManager 根据 stream type + CarAudioConfiguration 确定 output：
- `STREAM_MUSIC` → context `music` → `BUS00_MEDIA` → mixPort `media` → AUDIO_DEVICE_OUT_BUS

```cpp
// frameworks/av/services/audiopolicy/AudioPolicyManager.cpp
// getOutput() 根据 device 类型选择对应的 output handle
// 对于 BUS 设备，选择关联 media mixPort 的 output
```

#### 步骤3：AudioFlinger 打开输出流

```cpp
// frameworks/av/services/audioflinger/AudioFlinger.cpp

// 入口：AudioFlinger::openOutput()
status_t AudioFlinger::openOutput(audio_module_handle_t module,
                                  audio_io_handle_t *output,
                                  audio_config_t *config,
                                  const sp<DeviceDescriptorBase>& device,
                                  uint32_t *latencyMs,
                                  audio_output_flags_t flags)
{
    Mutex::Autolock _l(mLock);
    // 调用内部实现
    sp<ThreadBase> thread = openOutput_l(module, output, config,
                                          deviceType, address, flags);
    // 创建 MixerThread（普通PCM）或 OffloadThread（压缩offload）
    if (thread != 0) {
        // ...
    }
}

// 内部实现：openOutput_l()
sp<AudioFlinger::ThreadBase> AudioFlinger::openOutput_l(
    audio_module_handle_t module, audio_io_handle_t *output,
    audio_config_t *config, audio_devices_t deviceType,
    const String8& address, audio_output_flags_t flags)
{
    // 1. 分配唯一的 output handle
    *output = nextUniqueId(AUDIO_UNIQUE_ID_USE_OUTPUT);

    // 2. 调用 HAL 的 open_output_stream
    mHardwareStatus = AUDIO_HW_OUTPUT_OPEN;
    status_t status = outHwDev->open_output_stream(
            outHwDev, *output, devices, flags, config, &outputStream, address);

    // 3. 根据flags创建对应的 PlaybackThread
    if (flags & AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD) {
        thread = new OffloadThread(this, outputStream, *output, mSystemReady);
    } else if (flags & AUDIO_OUTPUT_FLAG_DIRECT) {
        thread = new DirectOutputThread(this, outputStream, *output, mSystemReady);
    } else {
        thread = new MixerThread(this, outputStream, *output, mSystemReady);
    }
    mPlaybackThreads.add(*output, thread);
    return thread;
}
```

#### 步骤4：Audio HAL 打开输出流

```c
// vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/audio_hw.c

int adev_open_output_stream(struct audio_hw_device *dev,
                            audio_io_handle_t handle,
                            audio_devices_t devices,
                            audio_output_flags_t flags,
                            struct audio_config *config,
                            struct audio_stream_out **stream_out,
                            const char *address)
{
    struct audio_device *adev = (struct audio_device *)dev;
    struct stream_out *out;

    out = (struct stream_out *)calloc(1, sizeof(struct stream_out));
    // ...

    // ★ Auto模式：通过 address 获取 car_audio_stream
    if (audio_extn_auto_hal_is_enabled()) {
        if (address != NULL) {
            out->car_audio_stream =
                auto_hal_get_car_audio_stream_from_address(address);
            // BUS00_MEDIA → CAR_AUDIO_STREAM_MEDIA (0x1)
        }
    }

    // ★ 根据设备类型确定 usecase
    if (devices & AUDIO_DEVICE_OUT_BUS) {
        // Auto 模式：car_audio_stream → usecase 映射
        ret = auto_hal_open_output_stream(out);
        // CAR_AUDIO_STREAM_MEDIA → USECASE_AUDIO_PLAYBACK_MEDIA
    }

    // 初始化 stream 操作函数
    out->stream.set_volume = out_set_volume;
    out->stream.write = out_write;
    // ...
}
```

```c
// vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/audio_extn/auto_hal.c

int auto_hal_open_output_stream(struct stream_out *out)
{
    switch(out->car_audio_stream) {
    case CAR_AUDIO_STREAM_MEDIA:
        out->usecase = USECASE_AUDIO_PLAYBACK_MEDIA;    // ★ 绑定usecase
        out->config = pcm_config_media;
        out->flags |= AUDIO_OUTPUT_FLAG_MEDIA;
        out->volume_l = out->volume_r = MAX_VOLUME_GAIN;
        break;
    case CAR_AUDIO_STREAM_SYS_NOTIFICATION:
        out->usecase = USECASE_AUDIO_PLAYBACK_SYS_NOTIFICATION;
        out->config = pcm_config_system;
        out->flags |= AUDIO_OUTPUT_FLAG_SYS_NOTIFICATION;
        break;
    // ... 其他 bus
    }
}
```

#### 步骤5：首次写入数据时启动输出流

```c
// vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/audio_hw.c

static ssize_t out_write(struct audio_stream_out *stream, const void *buffer,
                         size_t bytes)
{
    struct stream_out *out = (struct stream_out *)stream;

    if (out->standby) {
        out->standby = false;
        pthread_mutex_lock(&adev->lock);
        // ★ 启动输出流
        ret = start_output_stream(out);
        // ...
    }

    // ★ 写入 PCM 数据
    ret = pcm_write(out->pcm, (void *)buffer, bytes_to_write);
    // ...
}
```

```c
// vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/audio_hw.c

int start_output_stream(struct stream_out *out)
{
    struct audio_device *adev = out->dev;

    // ★ 1. 获取 PCM Device ID
    out->pcm_device_id = platform_get_pcm_device_id(out->usecase, PCM_PLAYBACK);
    // USECASE_AUDIO_PLAYBACK_MEDIA → MEDIA_PCM_DEVICE(0)

    // ★ 2. 创建 usecase 并加入列表
    uc_info = (struct audio_usecase *)calloc(1, sizeof(struct audio_usecase));
    uc_info->id = out->usecase;        // USECASE_AUDIO_PLAYBACK_MEDIA
    uc_info->type = PCM_PLAYBACK;
    uc_info->stream.out = out;
    list_add_tail(&adev->usecase_list, &uc_info->list);

    // ★ 3. 选择设备（路由配置 + ACDB校准）
    select_devices(adev, out->usecase);

    // ★ 4. 打开 PCM 设备节点
    out->pcm = pcm_open(adev->snd_card, out->pcm_device_id,
                         PCM_OUT, &out->config);
    // pcm_open(/dev/snd/pcmC0D0p, ...)

    // ★ 5. 发送 ACDB 校准数据到 ADSP
    platform_send_audio_calibration(adev->platform, uc_info,
                                     out->app_type_cfg.app_type);

    return 0;
}
```

#### 步骤6：select_devices 配置音频路由

```c
// vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/audio_hw.c

int select_devices(struct audio_device *adev, audio_usecase_t uc_id)
{
    snd_device_t out_snd_device = SND_DEVICE_NONE;
    struct audio_usecase *usecase = NULL;

    usecase = get_usecase_from_list(adev, uc_id);

    // ★ 1. 确定 snd_device
    // 对于 BUS 设备，通过 auto_hal 获取
    if (compare_device_type(&devices, AUDIO_DEVICE_OUT_BUS)) {
        out_snd_device = auto_hal_get_output_snd_device(adev, uc_id);
        // USECASE_AUDIO_PLAYBACK_MEDIA → SND_DEVICE_OUT_BUS_MEDIA
    }

    // ★ 2. 使能 snd_device（配置 mixer path + ACDB）
    enable_snd_device(adev, out_snd_device);
    // SND_DEVICE_OUT_BUS_MEDIA → "bus-speaker" mixer path
    // → audio_route_apply_and_update_path(adev->audio_route, device_name)

    // ★ 3. 使能音频路由（FE→BE mixer 连接）
    enable_audio_route(adev, usecase);
    // USECASE_AUDIO_PLAYBACK_MEDIA + SND_DEVICE_OUT_BUS_MEDIA
    // → mixer path: "deep-buffer-playback bus-speaker"
    // → 设置 "TERT_TDM_RX_0 Audio Mixer MultiMedia1" = 1
}
```

```c
// vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/audio_hw.c

int enable_snd_device(struct audio_device *adev, snd_device_t snd_device)
{
    char device_name[DEVICE_NAME_MAX_SIZE] = {0};

    // 获取设备名
    platform_get_snd_device_name(snd_device, device_name);
    // SND_DEVICE_OUT_BUS_MEDIA → "bus-speaker"

    // ★ 应用 mixer path（使能 BE 设备）
    audio_route_apply_and_update_path(adev->audio_route, device_name);

    return 0;
}

int enable_audio_route(struct audio_device *adev, struct audio_usecase *usecase)
{
    char mixer_path[MIXER_PATH_MAX_LENGTH];

    // 拼接 mixer path: usecase名 + 设备后缀
    // "deep-buffer-playback" + " " + "bus-speaker"
    strlcpy(mixer_path, use_case_table[usecase->id], sizeof(mixer_path));
    platform_add_backend_name(mixer_path, snd_device, usecase);

    // ★ 应用 FE→BE mixer 路由
    audio_route_apply_and_update_path(adev->audio_route, mixer_path);
    // 设置 "TERT_TDM_RX_0 Audio Mixer MultiMedia1" = 1

    return 0;
}
```

---

### 10.3 调节音量

#### 方式一：CarService 通过 AudioPort 设置增益（汽车场景主路径）

```java
// packages/services/Car/car-lib/src/android/car/media/CarAudioManager.java
// CarService 通过 AudioPort gain 控制 BUS 设备音量
carAudioManager.setGroupVolume(zoneId, groupId, index, flags);
```

```c
// vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/audio_extn/auto_hal.c

int auto_hal_set_audio_port_config(struct audio_hw_device *dev,
                        const struct audio_port_config *config)
{
    // 限制只对 BUS 设备支持 gain
    if (config->ext.device.type == AUDIO_DEVICE_OUT_BUS) {
        // millibel → 线性增益转换
        // q13 = (10^(mdb/100/20)) * (2^13)
        if (config->gain.values[0] <= (MIN_VOLUME_VALUE_MB + STEP_VALUE_MB))
            volume = MIN_VOLUME_GAIN;
        else
            volume = powf(10.0f, ((float)config->gain.values[0] / 2000));

        // ★ 设置到输出流的 volume
        out_ctxt->output->stream.set_volume(
            &out_ctxt->output->stream, volume, volume);
    }
}
```

#### 方式二：out_set_volume（流级别音量）

```c
// vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/audio_hw.c

static int out_set_volume(struct audio_stream_out *stream, float left,
                          float right)
{
    struct stream_out *out = (struct stream_out *)stream;

    ALOGD("%s: left_vol=%f, right_vol=%f", __func__, left, right);

    if (out->usecase == USECASE_AUDIO_PLAYBACK_MULTI_CH) {
        out->muted = (left == 0.0f);
    } else if (is_offload_usecase(out->usecase)) {
        // ★ BUS 设备的 offload 流音量覆盖逻辑
        if (compare_device_type(&out->device_list, AUDIO_DEVICE_OUT_BUS) &&
            (out->car_audio_stream == CAR_AUDIO_STREAM_MEDIA)) {
            // 使用对应 BUS 的 deep-buffer stream 的音量
            // ...
        }
    }
}
```

#### 方式三：PCM Playback Volume（Mixer 控制音量）

```c
// vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/audio_hw.c

static int out_set_pcm_volume(struct audio_stream_out *stream, float left,
                              float right)
{
    struct stream_out *out = (struct stream_out *)stream;
    char mixer_ctl_name[128];
    struct mixer_ctl *ctl;

    // ★ 通过 ALSA mixer 控制设置 FE PCM 音量
    int pcm_device_id = platform_get_pcm_device_id(out->usecase, PCM_PLAYBACK);
    snprintf(mixer_ctl_name, sizeof(mixer_ctl_name),
             "Playback %d Volume", pcm_device_id);
    // 例如: "Playback 0 Volume" (Media PCM device 0)

    ctl = mixer_get_ctl_by_name(adev->mixer, mixer_ctl_name);
    int volume = (int)(left * PCM_PLAYBACK_VOLUME_MAX);
    mixer_ctl_set_value(ctl, 0, volume);
    // ★ 直接写入 ALSA mixer control → Kernel ALSA driver → ADSP

    return 0;
}
```

---

### 10.4 读写音频数据

#### 写入数据：out_write → pcm_write

```c
// vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/audio_hw.c

static ssize_t out_write(struct audio_stream_out *stream, const void *buffer,
                         size_t bytes)
{
    struct stream_out *out = (struct stream_out *)stream;
    struct audio_device *adev = out->dev;
    ssize_t ret = 0;

    // ★ 1. 首次写入时启动输出流
    if (out->standby) {
        out->standby = false;
        pthread_mutex_lock(&adev->lock);
        ret = start_output_stream(out);  // 打开PCM + 配置路由
        pthread_mutex_unlock(&adev->lock);
    }

    // ★ 2. 写入 PCM 数据到驱动
    if (out->pcm) {
        if (out->hal_op_format != out->hal_ip_format &&
            out->convert_buffer != NULL) {
            // 格式转换后写入
            memcpy_by_audio_format(out->convert_buffer,
                                   out->hal_op_format,
                                   buffer, out->hal_ip_format,
                                   out->config.period_size * out->config.channels);
            ret = pcm_write(out->pcm, out->convert_buffer, ...);
        } else {
            // ★ 直接写入 PCM 数据
            ret = pcm_write(out->pcm, (void *)buffer, bytes_to_write);
        }
    }

    return ret;
}
```

#### pcm_write 内部实现（tinyalsa）

```c
// external/tinyalsa/pcm.c

int pcm_write(struct pcm *pcm, const void *data, unsigned int count)
{
    // ★ 通过 ioctl 写入 ALSA 驱动
    if (pcm->flags & PCM_MMAP) {
        return pcm_mmap_write(pcm, data, count);
    } else {
        // 标准 write() 系统调用
        // 写入 /dev/snd/pcmC0D0p
        if (write(pcm->fd, data, count) != count)
            return -EIO;
    }
    return 0;
}
```

#### 读取数据：in_read → pcm_read（录音场景）

```c
// vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/audio_hw.c

static ssize_t in_read(struct audio_stream_in *stream, void *buffer,
                       size_t bytes)
{
    struct stream_in *in = (struct stream_in *)stream;

    // ★ 首次读取时启动输入流
    if (in->standby) {
        in->standby = false;
        ret = start_input_stream(in);
    }

    // ★ 从 PCM 设备读取数据
    if (in->pcm) {
        ret = pcm_read(in->pcm, buffer, bytes);
    }

    return ret;
}
```

---

### 10.5 ADSP 通信流程

#### ACDB 校准数据下发

```c
// vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/msm8974/platform.c

int platform_send_audio_calibration(void *platform, struct audio_usecase *usecase,
                                    int app_type)
{
    struct platform_data *my_data = (struct platform_data *)platform;

    // ★ 1. 获取 snd_device 对应的 ACDB ID
    if (usecase->type == PCM_PLAYBACK)
        snd_device = usecase->out_snd_device;  // SND_DEVICE_OUT_BUS_MEDIA

    acdb_dev_id = acdb_device_table[platform_get_spkr_prot_snd_device(snd_device)];
    // SND_DEVICE_OUT_BUS_MEDIA → ACDB ID 60

    // ★ 2. 通过 ACDB loader 将校准数据发送到 ADSP
    // 包含：topology ID、app_type、sample_rate、ACDB device ID
    for (i = 0; i < num_devices; i++) {
        acdb_dev_id = acdb_device_table[new_snd_device[i]];
        platform_get_codec_backend_cfg(my_data->adev, new_snd_device[i], &backend_cfg);

        // ★ 发送校准到 ADSP (通过 ioctl → kernel → APR → ADSP)
        my_data->acdb_send_custom_toplogy_dev_cfg(acdb_dev_id,
                                                    app_type, sample_rate,
                                                    backend_cfg);
        my_data->acdb_send_cal(acdb_dev_id, app_type, sample_rate);
    }

    return 0;
}
```

#### ADSP 通信协议栈

```
┌──────────────────────────────┐
│  Audio HAL (userspace)       │
│  acdb_send_cal()             │
└──────────────┬───────────────┘
               │ ioctl(/dev/acdb_dev)
               ▼
┌──────────────────────────────┐
│  ACDB Kernel Driver          │
│  acdb_ioctl()                │
└──────────────┬───────────────┘
               │ APR (Async Port Router)
               ▼
┌──────────────────────────────┐
│  ADSP (Audio DSP)            │
│  ASM → topology → 处理链     │
│  → ADM → AFE → TDM输出      │
└──────────────┬───────────────┘
               │ I2S/TDM
               ▼
┌──────────────────────────────┐
│  Codec/Amplifier → Speaker   │
└──────────────────────────────┘
```

---

### 10.6 完整流程时序（以 Media 播放为例）

```
App            CarService     AudioPolicy     AudioFlinger     HAL            Driver        ADSP
 │                │               │               │             │               │            │
 │ AudioTrack()   │               │               │             │               │            │
 │───────────────>│               │               │             │               │            │
 │                │ getOutput()   │               │             │               │            │
 │                │──────────────>│               │             │               │            │
 │                │  BUS00_MEDIA  │               │             │               │            │
 │                │<──────────────│               │             │               │            │
 │                │               │ openOutput()  │             │               │            │
 │                │               │──────────────>│             │               │            │
 │                │               │               │ open_output │               │            │
 │                │               │               │────────────>│               │            │
 │                │               │               │             │ adev_open_    │            │
 │                │               │               │             │ output_stream │            │
 │                │               │               │             │ (address=     │            │
 │                │               │               │             │  BUS00_MEDIA) │            │
 │                │               │               │             │               │            │
 │ write(data)   │               │               │             │               │            │
 │──────────────>│               │               │             │               │            │
 │                │               │               │ out_write() │               │            │
 │                │               │               │────────────>│               │            │
 │                │               │               │             │ start_output_ │            │
 │                │               │               │             │ stream()      │            │
 │                │               │               │             │               │            │
 │                │               │               │             │ select_devices│            │
 │                │               │               │             │───────────────│            │
 │                │               │               │             │ enable_snd_   │            │
 │                │               │               │             │ device()      │            │
 │                │               │               │             │ "bus-speaker" │            │
 │                │               │               │             │──────────────>│            │
 │                │               │               │             │ enable_audio_ │            │
 │                │               │               │             │ route()       │            │
 │                │               │               │             │ MM1→TERT_TDM  │            │
 │                │               │               │             │──────────────>│            │
 │                │               │               │             │               │            │
 │                │               │               │             │ pcm_open()    │            │
 │                │               │               │             │ /dev/snd/     │            │
 │                │               │               │             │ pcmC0D0p      │            │
 │                │               │               │             │──────────────>│            │
 │                │               │               │             │               │            │
 │                │               │               │             │ send_acdb_cal │            │
 │                │               │               │             │ ACDB_ID=60    │            │
 │                │               │               │             │───────────────────────────>│
 │                │               │               │             │               │  topology  │
 │                │               │               │             │               │  加载      │
 │                │               │               │             │               │            │
 │                │               │               │             │ pcm_write()   │            │
 │                │               │               │             │ audio data    │            │
 │                │               │               │             │──────────────>│  FE→BE    │
 │                │               │               │             │               │───────────>│
 │                │               │               │             │               │  DSP处理  │
 │                │               │               │             │               │  →TDM输出 │
 │                │               │               │             │               │───────────>│
 │                │               │               │             │               │            │
 │                │               │               │             │               │     ┌──────┴──────┐
 │                │               │               │             │               │     │ Amp→Speaker│
 │                │               │               │             │               │     └─────────────┘
```

---

### 10.7 关键函数索引

| 流程阶段 | 函数 | 源文件 | 行号 |
|---|---|---|---|
| Framework: 打开输出 | `AudioFlinger::openOutput_l()` | `frameworks/av/services/audioflinger/AudioFlinger.cpp` | 2661 |
| HAL: 打开输出流 | `adev_open_output_stream()` | `vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/audio_hw.c` | 7724 |
| HAL: Auto流分配 | `auto_hal_open_output_stream()` | `vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/audio_extn/auto_hal.c` | 312 |
| HAL: 启动输出 | `start_output_stream()` | `vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/audio_hw.c` | 3818 |
| HAL: 设备选择 | `select_devices()` | `vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/audio_hw.c` | 2669 |
| HAL: 使能snd设备 | `enable_snd_device()` | `vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/audio_hw.c` | 1365 |
| HAL: 使能音频路由 | `enable_audio_route()` | `vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/audio_hw.c` | 1184 |
| HAL: PCM写入 | `out_write()` | `vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/audio_hw.c` | 5792 |
| HAL: PCM底层写 | `pcm_write()` | `external/tinyalsa/pcm.c` | - |
| HAL: 音量设置 | `out_set_volume()` | `vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/audio_hw.c` | 5570 |
| HAL: PCM音量 | `out_set_pcm_volume()` | `vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/audio_hw.c` | 5538 |
| HAL: Auto端口增益 | `auto_hal_set_audio_port_config()` | `vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/audio_extn/auto_hal.c` | 440 |
| HAL: Auto输出设备 | `auto_hal_get_output_snd_device()` | `vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/audio_extn/auto_hal.c` | 752 |
| Platform: ACDB校准 | `platform_send_audio_calibration()` | `vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/msm8974/platform.c` | 4989 |
| Platform: PCM ID | `platform_get_pcm_device_id()` | `vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/msm8974/platform.c` | - |
| Platform: 设备名 | `platform_get_snd_device_name()` | `vendor/qcom/opensource/audio-hal-ar/primary-hal/hal/msm8974/platform.c` | - |
| Mixer: 应用路由 | `audio_route_apply_and_update_path()` | `external/tinyalsa/mixer.c` | - |

---

## 11. snd_device 到 mixer_path 的映射机制

> 梳理 HAL 中 `snd_device` 如何一步步映射到 `mixer_paths_adp.xml` 中的具体 path，以及 ALSA mixer 控制的生效过程。

### 11.1 总体流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  select_devices()                                                          │
│    ├── 1. enable_snd_device(snd_device)       ← 阶段一：激活设备路径       │
│    │     ├── device_table[snd_device] → device_name (如 "bus-speaker")     │
│    │     └── audio_route_apply_and_update_path(device_name)                │
│    │         → 激活 mixer_paths_adp.xml 中的 <path name="bus-speaker">    │
│    │                                                                        │
│    └── 2. enable_audio_route(usecase)         ← 阶段二：激活音频路由       │
│          ├── use_case_table[usecase->id] → FE path (如 "media-playback")   │
│          ├── platform_add_backend_name() → 追加 BE 后缀                    │
│          │     └── backend_tag_table[snd_device] → suffix (Auto场景为NULL) │
│          └── audio_route_apply_and_update_path(mixer_path)                 │
│              → 激活 mixer_paths_adp.xml 中的 <path name="media-playback"> │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 11.2 阶段一：enable_snd_device() — 激活设备路径

**代码位置**：`audio_hw.c:1365`

```c
int enable_snd_device(struct audio_device *adev, snd_device_t snd_device)
{
    char device_name[DEVICE_NAME_MAX_SIZE] = {0};

    // Step 1: 通过 device_table[] 查找 snd_device 对应的 mixer device name
    if (platform_get_snd_device_name_extn(adev->platform, snd_device, device_name) < 0) {
        ALOGE("%s: Invalid sound device returned", __func__);
        return -EINVAL;
    }

    // Step 2: 应用 mixer_path 中名为 device_name 的 path
    ALOGD("%s: snd_device(%d: %s)", __func__, snd_device, device_name);
    audio_route_apply_and_update_path(adev->audio_route, device_name);
    // ...
}
```

#### device_table[] 映射（snd_device → device name）

**代码位置**：`platform.c:596`

```c
/* device_table[] 定义在 platform.c，snd_device → mixer device name */
[SND_DEVICE_OUT_BUS_MEDIA] = "bus-speaker",
[SND_DEVICE_OUT_BUS_SYS]  = "bus-speaker",
[SND_DEVICE_OUT_BUS_NAV]  = "bus-speaker",
[SND_DEVICE_OUT_BUS_PHN]  = "bus-speaker",
[SND_DEVICE_OUT_BUS_RSE]  = "bus-speaker",

/* 对比手机平台 */
[SND_DEVICE_OUT_SPEAKER]              = "speaker",
[SND_DEVICE_OUT_HEADPHONES]           = "headphones",
[SND_DEVICE_OUT_SPEAKER_AND_HEADPHONES] = "speaker-and-headphones",
```

> Auto 平台所有 BUS 输出设备都映射到同一个 `"bus-speaker"`，因为底层物理输出通道（TDM）的选择不是在 device_table 层区分，而是在 enable_audio_route 阶段通过不同的 usecase path 来区分。

#### mixer_paths_adp.xml 中对应的 path

```xml
<path name="bus-speaker">
    <!--ctl name="TERT_TDM_RX_0 Channels" value="Six" /-->
</path>
```

> 注：`bus-speaker` path 在 adp 平台几乎是空的（TDM Channels 配置被注释掉），因为 BE 通道配置实际上由 `enable_audio_route()` 阶段的 usecase path 完成。`enable_snd_device()` 阶段仅做设备启用标记和 ACDB 校准。

### 11.3 阶段二：enable_audio_route() — 激活音频路由（FE + BE 拼接）

**代码位置**：`audio_hw.c:1184`

```c
int enable_audio_route(struct audio_device *adev, struct audio_usecase *usecase)
{
    snd_device_t snd_device;
    char mixer_path[MIXER_PATH_MAX_LENGTH];

    // 获取当前 usecase 对应的 snd_device
    snd_device = usecase->out_snd_device;

    // Step 1: 从 use_case_table[] 获取 FE (Front-End) path name
    strlcpy(mixer_path, use_case_table[usecase->id], sizeof(mixer_path));
    // USECASE_AUDIO_PLAYBACK_MEDIA          → "media-playback"
    // USECASE_AUDIO_PLAYBACK_SYS_NOTIFICATION → "sys-notification-playback"
    // USECASE_AUDIO_PLAYBACK_NAV_GUIDANCE   → "nav-guidance-playback"
    // USECASE_AUDIO_PLAYBACK_PHONE          → "phone-playback"
    // USECASE_AUDIO_PLAYBACK_FRONT_PASSENGER → "front-passenger-playback"
    // USECASE_AUDIO_PLAYBACK_REAR_SEAT      → "rear-seat-playback"

    // Step 2: 追加 BE (Back-End) 后缀
    platform_add_backend_name(mixer_path, snd_device, usecase);

    ALOGD("%s: apply mixer and update path: %s", __func__, mixer_path);

    // Step 3: 应用拼接后的 mixer_path
    audio_route_apply_and_update_path(adev->audio_route, mixer_path);
}
```

### 11.4 platform_add_backend_name() — BE 后缀拼接

**代码位置**：`platform.c:4156`

```c
void platform_add_backend_name(char *mixer_path, snd_device_t snd_device,
                               struct audio_usecase *usecase)
{
    const char * suffix = backend_tag_table[snd_device];

    if (suffix != NULL) {
        strlcat(mixer_path, " ", MIXER_PATH_MAX_LENGTH);
        strlcat(mixer_path, suffix, MIXER_PATH_MAX_LENGTH);
        // 例如手机平台：
        // mixer_path = "deep-buffer-playback" + " " + "speaker"
        //           = "deep-buffer-playback speaker"
    }
    // Auto 场景：backend_tag_table[SND_DEVICE_OUT_BUS_*] == NULL
    // 所以不追加任何后缀，mixer_path 保持为 "media-playback"
}
```

### 11.5 backend_tag_table[] 的填充

**代码位置**：`platform.c:1244`（声明）、`platform.c:2100`（初始化）

```c
static char * backend_tag_table[SND_DEVICE_MAX] = {0};
```

#### 硬编码条目（set_platform_defaults）

```c
/* 代码在 set_platform_defaults() 中硬编码，注释说明可被 XML 覆盖 */
// To overwrite these go to the audio_platform_info.xml file.
backend_tag_table[SND_DEVICE_OUT_BT_SCO]          = strdup("bt-sco");
backend_tag_table[SND_DEVICE_OUT_BT_SCO_WB]       = strdup("bt-sco-wb");
backend_tag_table[SND_DEVICE_OUT_HDMI]             = strdup("hdmi");
backend_tag_table[SND_DEVICE_OUT_DISPLAY_PORT]     = strdup("display-port");
backend_tag_table[SND_DEVICE_OUT_USB_HEADSET]      = strdup("usb-headset");
backend_tag_table[SND_DEVICE_OUT_BT_A2DP]          = strdup("bt-a2dp");
// ... 等等

/* 注意：SND_DEVICE_OUT_BUS_MEDIA/SYS/NAV/PHN/RSE 没有硬编码条目！ */
```

#### XML 覆盖（audio_platform_info.xml）

**文件**：`configs/msmnile_au/audio_platform_info.xml`

XML 解析最终调用 `platform_set_snd_device_backend()`（`platform.c:9485`）来覆盖表项：

```c
int platform_set_snd_device_backend(snd_device_t device, const char *backend_tag,
                                    const char *hw_interface)
{
    if (backend_tag != NULL) {
        if (backend_tag_table[device])
            free(backend_tag_table[device]);
        backend_tag_table[device] = strdup(backend_tag);  // 覆盖 backend_tag
    }

    if (hw_interface != NULL) {
        if (hw_interface_table[device])
            free(hw_interface_table[device]);
        hw_interface_table[device] = strdup(hw_interface);  // 覆盖 hw_interface
    }
}
```

但 `msmnile_au/audio_platform_info.xml` 中只配置了 `interface`，没有配置 `backend_tag`：

```xml
<device name="SND_DEVICE_OUT_BUS_MEDIA" interface="TERT_TDM_RX_0"/>
<device name="SND_DEVICE_OUT_BUS_SYS"  interface="TERT_TDM_RX_0"/>
<device name="SND_DEVICE_OUT_BUS_NAV"  interface="TERT_TDM_RX_1"/>
<device name="SND_DEVICE_OUT_BUS_PHN"  interface="TERT_TDM_RX_2"/>
<device name="SND_DEVICE_OUT_BUS_RSE"  interface="QUAT_TDM_RX_0"/>
```

> **结论**：Auto 场景下 `backend_tag_table[SND_DEVICE_OUT_BUS_*]` 全部为 NULL，`platform_add_backend_name()` 不会追加后缀，mixer_path 就是纯 FE path 名。

### 11.6 hw_interface_table[] — BE 硬件接口

**代码位置**：`platform.c:2282`

```c
hw_interface_table[SND_DEVICE_OUT_BUS_MEDIA] = strdup("TERT_TDM_RX_0");
hw_interface_table[SND_DEVICE_OUT_BUS_SYS]  = strdup("TERT_TDM_RX_0");
hw_interface_table[SND_DEVICE_OUT_BUS_NAV]  = strdup("TERT_TDM_RX_1");
hw_interface_table[SND_DEVICE_OUT_BUS_PHN]  = strdup("TERT_TDM_RX_2");
hw_interface_table[SND_DEVICE_OUT_BUS_RSE]  = strdup("QUAT_TDM_RX_0");
```

`hw_interface_table` **不参与 mixer_path 拼接**，它用于：
- `platform_check_backends_match()`：判断两个 snd_device 是否共享同一 BE 接口
- `platform_set_codec_backend_cfg()`：设置 BE 的采样率/位宽等参数
- BE DAI ID 查找（`platform_get_snd_device_backend_index()`）

### 11.7 audio_route_apply_and_update_path() — mixer path 的查找与执行

**代码位置**：`system/media/audio_route/audio_route.c:882`

```c
int audio_route_apply_and_update_path(struct audio_route *ar, const char *name)
{
    // Step 1: 精确匹配 mixer_paths XML 中的 <path name="xxx">
    if (audio_route_apply_path(ar, name) < 0) {
        ALOGE("unable to find path '%s'", name);
        return -1;
    }
    // Step 2: 更新 ALSA mixer 控制器（实际写入驱动）
    return audio_route_update_path(ar, name, DIRECTION_FORWARD);
}
```

内部查找逻辑（`path_get_by_name`，`audio_route.c:181`）：

```c
static struct mixer_path *path_get_by_name(struct audio_route *ar, const char *name)
{
    unsigned int i;
    for (i = 0; i < ar->num_mixer_paths; i++)
        if (strcmp(ar->mixer_path[i].name, name) == 0)   // 精确字符串匹配
            return &ar->mixer_path[i];
    return NULL;
}
```

> `audio_route` 在初始化时解析 `mixer_paths_adp.xml`，将所有 `<path>` 和 `<ctl>` 构建为内存数据结构。`apply_and_update_path()` 通过名字精确查找，然后遍历 path 中的所有 ctl 设置项，逐个写入 ALSA mixer。

### 11.8 mixer_paths_adp.xml 中 Auto 相关 path 完整定义

```xml
<!-- FE path: 各 usecase 对应的 Front-End 路由 -->
<path name="media-playback">
    <ctl name="TERT_TDM_RX_0 Channels" value="Six" />
    <ctl name="TERT_TDM_RX_0 Audio Mixer MultiMedia1" value="1" />
</path>

<path name="sys-notification-playback">
    <ctl name="TERT_TDM_RX_0 Channels" value="Six" />
    <ctl name="TERT_TDM_RX_0 Audio Mixer MultiMedia5" value="1" />
</path>

<path name="nav-guidance-playback">
    <ctl name="TERT_TDM_RX_1 Channels" value="One" />
    <ctl name="TERT_TDM_RX_1 Audio Mixer MultiMedia2" value="1" />
</path>

<path name="phone-playback">
    <ctl name="TERT_TDM_RX_2 Channels" value="One" />
    <ctl name="TERT_TDM_RX_2 Audio Mixer MultiMedia10" value="1" />
</path>

<path name="front-passenger-playback">
    <ctl name="QUAT_TDM_RX_0 Channels" value="Eight" />
    <ctl name="QUAT_TDM_RX_0 Audio Mixer MultiMedia23" value="1" />
</path>

<path name="rear-seat-playback">
    <ctl name="QUIN_TDM_RX_0 Channels" value="Sixteen" />
    <ctl name="QUIN_TDM_RX_0 Audio Mixer MultiMedia22" value="1" />
</path>

<!-- BE path: 设备层 path（Auto 场景几乎为空） -->
<path name="bus-speaker">
    <!--ctl name="TERT_TDM_RX_0 Channels" value="Six" /-->
</path>
```

每个 FE path 包含两个关键 mixer control：
1. **`XXX Channels`**：设置 BE TDM 通道数
2. **`XXX Audio Mixer MultiMediaN`**：将 FE PCM (MultiMediaN) 路由到 BE TDM (XXX)

### 11.9 完整映射表：snd_device → mixer_path

| snd_device | device_table[] | enable_snd_device path | usecase | use_case_table[] | backend_tag | 最终 mixer_path | mixer ctl 效果 |
|---|---|---|---|---|---|---|---|
| SND_DEVICE_OUT_BUS_MEDIA | "bus-speaker" | `bus-speaker` | USECASE_AUDIO_PLAYBACK_MEDIA | "media-playback" | NULL | **media-playback** | TERT_TDM_RX_0 Mixer MM1 |
| SND_DEVICE_OUT_BUS_SYS | "bus-speaker" | `bus-speaker` | USECASE_AUDIO_PLAYBACK_SYS_NOTIFICATION | "sys-notification-playback" | NULL | **sys-notification-playback** | TERT_TDM_RX_0 Mixer MM5 |
| SND_DEVICE_OUT_BUS_NAV | "bus-speaker" | `bus-speaker` | USECASE_AUDIO_PLAYBACK_NAV_GUIDANCE | "nav-guidance-playback" | NULL | **nav-guidance-playback** | TERT_TDM_RX_1 Mixer MM2 |
| SND_DEVICE_OUT_BUS_PHN | "bus-speaker" | `bus-speaker` | USECASE_AUDIO_PLAYBACK_PHONE | "phone-playback" | NULL | **phone-playback** | TERT_TDM_RX_2 Mixer MM10 |
| SND_DEVICE_OUT_BUS_PAX | "bus-speaker" | `bus-speaker` | USECASE_AUDIO_PLAYBACK_FRONT_PASSENGER | "front-passenger-playback" | NULL | **front-passenger-playback** | QUAT_TDM_RX_0 Mixer MM23 |
| SND_DEVICE_OUT_BUS_RSE | "bus-speaker" | `bus-speaker` | USECASE_AUDIO_PLAYBACK_REAR_SEAT | "rear-seat-playback" | NULL | **rear-seat-playback** | QUIN_TDM_RX_0 Mixer MM22 |

### 11.10 与手机平台的对比

手机平台走的是 **"FE名 + 空格 + BE后缀"** 的拼接模式：

```
use_case_table[USECASE_AUDIO_PLAYBACK_DEEP_BUFFER] = "deep-buffer-playback"
backend_tag_table[SND_DEVICE_OUT_SPEAKER] = NULL  (使用 default_rx_backend)
最终 mixer_path = "deep-buffer-playback speaker"
```

```
use_case_table[USECASE_AUDIO_PLAYBACK_DEEP_BUFFER] = "deep-buffer-playback"
backend_tag_table[SND_DEVICE_OUT_HEADPHONES] = NULL  (使用 default_rx_backend)
最终 mixer_path = "deep-buffer-playback headphones"
```

```
use_case_table[USECASE_AUDIO_PLAYBACK_DEEP_BUFFER] = "deep-buffer-playback"
backend_tag_table[SND_DEVICE_OUT_HDMI] = "hdmi"
最终 mixer_path = "deep-buffer-playback hdmi"
```

Auto 平台不需要这种拼接，因为**每个 usecase 已经在 mixer_paths_adp.xml 中直接指定了对应的 TDM BE**：

```xml
<!-- media-playback 已经包含 TERT_TDM_RX_0 的路由，不需要后缀 -->
<path name="media-playback">
    <ctl name="TERT_TDM_RX_0 Audio Mixer MultiMedia1" value="1" />
</path>
```

而手机平台通过后缀组合来复用同一个 FE path：

```xml
<!-- deep-buffer-playback 不含 BE 路由，需要与 "speaker" 或 "headphones" 组合 -->
<path name="deep-buffer-playback">
    <ctl name="PRI_MI2S_RX Audio Mixer MultiMedia1" value="1" />
</path>
<path name="deep-buffer-playback speaker">
    <ctl name="PRI_MI2S_RX Audio Mixer MultiMedia1" value="1" />
    <!-- + speaker 特定的配置 -->
</path>
```

### 11.11 关键数据结构关系图

```
┌──────────────────────────────────────────────────────────────────┐
│                     三张核心映射表                                 │
│                                                                  │
│  device_table[]          backend_tag_table[]       hw_interface_table[]  │
│  ┌─────────────────┐     ┌──────────────────┐     ┌──────────────────┐ │
│  │ snd_device →    │     │ snd_device →     │     │ snd_device →     │ │
│  │ device_name     │     │ backend suffix   │     │ HW interface     │ │
│  ├─────────────────┤     ├──────────────────┤     ├──────────────────┤ │
│  │ BUS_MEDIA →     │     │ BUS_MEDIA → NULL │     │ BUS_MEDIA →      │ │
│  │  "bus-speaker"  │     │ BUS_SYS  → NULL │     │  TERT_TDM_RX_0   │ │
│  │ BUS_SYS →       │     │ BUS_NAV  → NULL │     │ BUS_SYS →        │ │
│  │  "bus-speaker"  │     │ BUS_PHN  → NULL │     │  TERT_TDM_RX_0   │ │
│  │ BUS_NAV →       │     │ BUS_RSE  → NULL │     │ BUS_NAV →        │ │
│  │  "bus-speaker"  │     │ HDMI → "hdmi"   │     │  TERT_TDM_RX_1   │ │
│  │ BUS_PHN →       │     │ BT_SCO →"bt-sco"│     │ BUS_PHN →        │ │
│  │  "bus-speaker"  │     │ USB →"usb-heads"│     │  TERT_TDM_RX_2   │ │
│  │ BUS_RSE →       │     │                  │     │ BUS_RSE →        │ │
│  │  "bus-speaker"  │     │                  │     │  QUAT_TDM_RX_0   │ │
│  └────────┬────────┘     └────────┬─────────┘     └────────┬─────────┘ │
│           │                       │                         │          │
│           ▼                       ▼                         ▼          │
│   enable_snd_device()    enable_audio_route()    platform_check_      │
│   激活设备 path          拼接 BE 后缀            backends_match()     │
│   (阶段一)               (阶段二)                BE 配置决策           │
│                                                   codec_backend_cfg  │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                    use_case_table[]                              │
│  ┌──────────────────────────────────────────────┐               │
│  │ usecase → FE path name                       │               │
│  ├──────────────────────────────────────────────┤               │
│  │ PLAYBACK_MEDIA          → "media-playback"             │    │
│  │ PLAYBACK_SYS_NOTIFICATION→ "sys-notification-playback" │    │
│  │ PLAYBACK_NAV_GUIDANCE   → "nav-guidance-playback"     │    │
│  │ PLAYBACK_PHONE          → "phone-playback"            │    │
│  │ PLAYBACK_FRONT_PASSENGER→ "front-passenger-playback"  │    │
│  │ PLAYBACK_REAR_SEAT      → "rear-seat-playback"        │    │
│  └──────────────────┬───────────────────────────┘               │
│                     ▼                                            │
│          enable_audio_route()                                    │
│          mixer_path 基础名 (阶段二)                              │
└──────────────────────────────────────────────────────────────────┘
```

### 11.12 关键函数索引

| 函数 | 文件 | 行号 | 作用 |
|---|---|---|---|
| `enable_snd_device()` | `audio_hw.c` | 1365 | snd_device→device_name→激活设备path |
| `enable_audio_route()` | `audio_hw.c` | 1184 | usecase→FE path→拼接BE后缀→激活路由path |
| `disable_audio_route()` | `audio_hw.c` | 1286 | 禁用音频路由（reset path） |
| `platform_add_backend_name()` | `platform.c` | 4156 | 查 backend_tag_table 追加 BE 后缀到 mixer_path |
| `platform_get_snd_device_name_extn()` | `platform.c` | 4135 | 查 device_table 获取 device_name |
| `platform_set_snd_device_backend()` | `platform.c` | 9485 | XML 解析时覆盖 backend_tag_table 和 hw_interface_table |
| `platform_get_snd_device_backend_interface()` | `platform.c` | 9520 | 查 hw_interface_table 获取 BE 接口名 |
| `audio_route_apply_and_update_path()` | `audio_route.c` | 882 | 精确查找 path 名并写入 ALSA mixer |
| `path_get_by_name()` | `audio_route.c` | 181 | 在内存中精确匹配 path 名 |
| `set_platform_defaults()` | `platform.c` | 2100 | 初始化 backend_tag_table/hw_interface_table 默认值 |

---

## 12. pcm_write 到 ADSP 的数据传输机制

> 梳理 `pcm_write()` 写入的音频数据如何经过内核 ALSA 子系统、APR 通信，最终到达 ADSP 进行处理的全过程。核心结论：**数据不是由驱动 push 给 ADSP，而是写入共享内存后通知 ADSP 主动拉取**。

### 12.1 数据流总览

```
┌──────────────────────────────────────────────────────────────────┐
│ 用户空间 (AudioFlinger / HAL)                                    │
│   out_write() → pcm_write(data)                                  │
│     → ioctl(SNDRV_PCM_IOCTL_WRITEI_FRAMES)                      │
│     → copy_from_user() → 数据写入 DMA 共享内存                   │
└──────────────────────────────┬───────────────────────────────────┘
                               │ (数据已在共享内存中)
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│ 内核空间 (q6asm_dai ASoC driver)                                 │
│   event_handler / trigger                                        │
│     → q6asm_write_async()                                        │
│       → 构造 ASM_DATA_CMD_WRITE_V2 (buf_phys_addr, len)         │
│       → apr_send_pkt() ──────────────────────┐                  │
└───────────────────────────────────────────────┼──────────────────┘
                                                │ APR (glancer/spmi)
                                                ▼
┌──────────────────────────────────────────────────────────────────┐
│ ADSP (Audio DSP)                                                 │
│   ASM (Audio Stream Manager)                                     │
│     ← 收到 ASM_DATA_CMD_WRITE_V2                                │
│     → 根据 buf_phys_addr 从共享内存 DMA 读取音频数据             │
│     → ASM 处理 (topology: 解码/音效/混音)                        │
│     → 通过 ADM 路由到 AFE port (如 TERT_TDM_RX_0)               │
│   AFE (Audio Front End)                                          │
│     → 将数据通过 TDM 总线发送到 Codec/DAC                        │
│   ASM → APR 返回 ASM_DATA_EVENT_WRITE_DONE_V2                   │
│     → 内核收到回调 → snd_pcm_period_elapsed()                    │
│     → 用户空间可以写入下一个 period                               │
└──────────────────────────────────────────────────────────────────┘
```

### 12.2 第一步：pcm_write() — 数据写入共享内存

**代码位置**：`vendor/qcom/opensource/tinyalsa/pcm.c:534`

```c
int pcm_write(struct pcm *pcm, const void *data, unsigned int count)
{
    struct snd_xferi x;

    x.buf = (void*)data;
    x.frames = count / (pcm->config.channels *
                        pcm_format_to_bits(pcm->config.format) / 8);

    for (;;) {
        if (!pcm->running) {
            int prepare_error = pcm_prepare(pcm);
            if (prepare_error)
                return prepare_error;
            // 首次写入：通过 ioctl 将数据从用户空间拷贝到内核 DMA buffer
            if (pcm->ops->ioctl(pcm->data, SNDRV_PCM_IOCTL_WRITEI_FRAMES, &x))
                return oops(pcm, errno, "cannot write initial data");
            pcm->running = 1;
            return 0;
        }
        // 后续写入
        if (pcm->ops->ioctl(pcm->data, SNDRV_PCM_IOCTL_WRITEI_FRAMES, &x)) {
            // ...
        }
        return 0;
    }
}
```

`SNDRV_PCM_IOCTL_WRITEI_FRAMES` 在内核中的处理链：

```
snd_pcm_playback_ioctl1()          // pcm_native.c
  → snd_pcm_xferi_frames_ioctl()   // pcm_native.c:2842
    → snd_pcm_lib_write()          // pcm_lib.c
      → default_write_copy()       // pcm_lib.c:1926
        → copy_from_user(runtime->dma_area + hwoff, buf, bytes)
```

**关键**：`runtime->dma_area` 是 ALSA 分配的 DMA buffer，这是一块 **CPU 和 ADSP 共享的物理连续内存**。

### 12.3 DMA 共享内存的分配与映射

#### DMA buffer 分配

**代码位置**：`kernel/msm-5.4/sound/soc/qcom/qdsp6/q6asm-dai.c:325`

```c
static int q6asm_dai_open(struct snd_pcm_substream *substream)
{
    // 分配 audio client（包含与 ADSP 通信的 session）
    prtd->audio_client = q6asm_audio_client_alloc(dev,
                (q6asm_cb)event_handler, prtd, stream_id,
                LEGACY_PCM_MODE);

    // 设置 DMA buffer
    runtime->dma_bytes = q6asm_dai_hardware_playback.buffer_bytes_max;

    // 获取 DMA buffer 的物理地址（ADSP 需要物理地址来访问）
    if (pdata->sid < 0)
        prtd->phys = substream->dma_buffer.addr;
    else
        prtd->phys = substream->dma_buffer.addr | (pdata->sid << 32);

    snd_pcm_set_runtime_buffer(substream, &substream->dma_buffer);
}
```

#### 共享内存映射到 ADSP 地址空间

**代码位置**：`kernel/msm-5.4/sound/soc/qcom/qdsp6/q6asm.c:423`

`prepare` 阶段调用 `q6asm_map_memory_regions()`，将 DMA buffer 的物理地址注册到 ADSP：

```c
int q6asm_map_memory_regions(unsigned int dir, struct audio_client *ac,
                             phys_addr_t phys,
                             size_t period_sz, unsigned int periods)
{
    // 记录每个 period 的物理地址
    buf[0].phys = phys;
    buf[0].size = period_sz;
    for (cnt = 1; cnt < periods; cnt++) {
        buf[cnt].phys = buf[0].phys + (cnt * period_sz);
        buf[cnt].size = period_sz;
    }

    // 通过 APR 向 ADSP 发送内存映射命令
    rc = __q6asm_memory_map_regions(ac, dir, period_sz, periods, 1);
}
```

`__q6asm_memory_map_regions()` 发送 APR 命令：

```c
static int __q6asm_memory_map_regions(struct audio_client *ac, int dir, ...)
{
    // 构造 APR 包
    pkt->hdr.opcode = ASM_CMD_SHARED_MEM_MAP_REGIONS;
    cmd->mem_pool_id = ADSP_MEMORY_MAP_SHMEM8_4K_POOL;
    cmd->num_regions = num_regions;

    // 填入共享内存的物理地址和大小
    mregions->shm_addr_lsw = lower_32_bits(ab->phys);
    mregions->shm_addr_msw = upper_32_bits(ab->phys);
    mregions->mem_size_bytes = buf_sz;

    // 发送给 ADSP
    rc = q6asm_apr_send_session_pkt(a, ac, pkt,
                    ASM_CMDRSP_SHARED_MEM_MAP_REGIONS);
}
```

> ADSP 收到后，将该物理内存区域映射到自己的地址空间，后续 ADSP 就可以通过物理地址直接 DMA 读取这块内存中的音频数据。

### 12.4 第二步：q6asm_write_async() — 通知 ADSP 读取数据

**代码位置**：`kernel/msm-5.4/sound/soc/qcom/qdsp6/q6asm.c:1205`

驱动并不搬运音频数据本身，而是发送一个**命令包**，告诉 ADSP 数据的物理地址和大小：

```c
int q6asm_write_async(struct audio_client *ac, uint32_t len,
                      uint32_t msw_ts, uint32_t lsw_ts, uint32_t wflags)
{
    struct asm_data_cmd_write_v2 *write;
    struct audio_buffer *ab;

    // 获取当前 period buffer 的物理地址
    ab = &port->buf[port->dsp_buf];

    // 构造 ASM 写命令
    pkt->hdr.opcode = ASM_DATA_CMD_WRITE_V2;        // ASM 写命令码
    write->buf_addr_lsw = lower_32_bits(ab->phys);   // buffer 物理地址 (低32位)
    write->buf_addr_msw = upper_32_bits(ab->phys);   // buffer 物理地址 (高32位)
    write->buf_size = len;                            // 数据大小 (字节)
    write->seq_id = port->dsp_buf;                    // 序列号
    write->timestamp_lsw = lsw_ts;                    // 时间戳
    write->timestamp_msw = msw_ts;
    write->mem_map_handle =                            // 内存映射句柄
        ac->port[SNDRV_PCM_STREAM_PLAYBACK].mem_map_handle;

    // 通过 APR 发送命令给 ADSP ASM
    rc = apr_send_pkt(ac->adev, pkt);

    // 移动到下一个 period buffer
    port->dsp_buf++;
    if (port->dsp_buf >= port->num_periods)
        port->dsp_buf = 0;  // 环形缓冲区
}
```

**核心**：`ASM_DATA_CMD_WRITE_V2` 命令携带的是 buffer 的**物理地址和大小**，不携带音频数据本身。ADSP 收到命令后，自己通过 DMA 从该物理地址读取数据。

### 12.5 第三步：ADSP 内部处理

ADSP 收到 `ASM_DATA_CMD_WRITE_V2` 后的处理流程：

```
ADSP ASM 收到 ASM_DATA_CMD_WRITE_V2
  │
  ├── 1. 根据 mem_map_handle 找到已映射的共享内存区域
  ├── 2. 根据 buf_addr 从共享内存 DMA 读取音频 PCM 数据
  ├── 3. ASM 执行 topology 处理：
  │     ├── 解码（如果需要）
  │     ├── 音效处理（Bass Boost / Virtual Surround 等，由 ACDB topology 决定）
  │     └── 混音（多路流合并）
  ├── 4. 将处理后的数据发送到 ADM (Audio Device Manager)
  │     └── ADM 根据路由将数据送到对应的 AFE port
  └── 5. AFE (Audio Front End) 将数据输出到物理接口
        └── TERT_TDM_RX_0 → TDM 总线 → 外部 DAC/Codec → 喇叭
```

AFE port 的启动通过 `q6afe_port_start()` 实现：

```c
// kernel/msm-5.4/sound/soc/qcom/qdsp6/q6afe.c:1306
int q6afe_port_start(struct q6afe_port *port)
{
    // 先设置 AFE port 参数（TDM 配置：通道数/位宽/采样率等）
    ret = q6afe_port_set_param_v2(port, &port->port_cfg, param_id,
                                   AFE_MODULE_AUDIO_DEV_INTERFACE,
                                   sizeof(port->port_cfg));

    // 发送 AFE 启动命令
    pkt->hdr.opcode = AFE_PORT_CMD_DEVICE_START;
    start->port_id = port_id;  // 如 TERT_TDM_RX_0 的 port_id
    ret = afe_apr_send_pkt(afe, pkt, port);
}
```

### 12.6 第四步：ADSP 回调通知 — WRITE_DONE

ADSP 处理完一个 period 的数据后，通过 APR 返回 `ASM_DATA_EVENT_WRITE_DONE_V2`：

**内核回调处理**（`kernel/msm-5.4/sound/soc/qcom/qdsp6/q6asm.c:593`）：

```c
// q6asm 的 APR 回调
case ASM_DATA_EVENT_WRITE_DONE_V2:
    client_event = ASM_CLIENT_EVENT_DATA_WRITE_DONE;
    // 验证 buffer 物理地址是否匹配
    if (lower_32_bits(phys) != result->opcode ||
        upper_32_bits(phys) != result->status) {
        dev_err(ac->dev, "Expected addr %pa\n", &port->buf[hdr->token].phys);
        ret = -EINVAL;
        goto done;
    }
    break;
```

**q6asm_dai 事件处理**（`kernel/msm-5.4/sound/soc/qcom/qdsp6/q6asm-dai.c:173`）：

```c
static void event_handler(uint32_t opcode, uint32_t token,
                          void *payload, void *priv)
{
    struct q6asm_dai_rtd *prtd = priv;
    struct snd_pcm_substream *substream = prtd->substream;

    switch (opcode) {
    case ASM_CLIENT_EVENT_CMD_RUN_DONE:
        // 首次启动：发送第一个 write 命令
        q6asm_write_async(prtd->audio_client,
                   prtd->pcm_count, 0, 0, NO_TIMESTAMP);
        break;

    case ASM_CLIENT_EVENT_DATA_WRITE_DONE:
        // ADSP 已消费一个 period
        prtd->pcm_irq_pos += prtd->pcm_count;
        snd_pcm_period_elapsed(substream);  // 通知 ALSA core：period 可用
        if (prtd->state == Q6ASM_STREAM_RUNNING)
            // 通知 ADSP 读取下一个 period
            q6asm_write_async(prtd->audio_client,
                       prtd->pcm_count, 0, 0, NO_TIMESTAMP);
        break;
    }
}
```

### 12.7 环形缓冲区交互时序

```
时间轴 ──────────────────────────────────────────────────────────→

用户空间:  [write period0] [write period1] [write period2] ...
               │                │                │
               ▼                ▼                ▼
DMA Buffer: [P0已写入]    [P1已写入]    [P2已写入] ...
               │                              │
               ▼                              ▼
驱动通知:  q6asm_write_async(P0)     q6asm_write_async(P2) ...
ADSP读取:  [读P0]           [读P1]           [读P2] ...
               │                │                │
               ▼                ▼                ▼
ADSP回调:  WRITE_DONE(P0)  WRITE_DONE(P1)  WRITE_DONE(P2) ...
               │                │                │
               ▼                ▼                ▼
ALSA:      period_elapsed() period_elapsed() period_elapsed()
               │                │                │
               ▼                ▼                ▼
用户空间:  可写新P0        可写新P1        可写新P2 ...
```

### 12.8 关键结论

| 问题 | 回答 |
|------|------|
| pcm_write 是否直接把数据传给 ADSP？ | **不是**。pcm_write 只做 `copy_from_user`，把数据写入 DMA 共享内存 |
| 谁把数据传给 ADSP？ | **ADSP 自己从共享内存 DMA 拉取**，驱动只发送"通知命令" |
| 驱动发送给 ADSP 的是什么？ | `ASM_DATA_CMD_WRITE_V2` 命令包，包含 buffer **物理地址**和大小，不包含音频数据 |
| ADSP 如何访问共享内存？ | `prepare` 阶段通过 `ASM_CMD_SHARED_MEM_MAP_REGIONS` 将物理内存映射到 ADSP 地址空间 |
| ADSP 处理完如何通知？ | 通过 APR 返回 `ASM_DATA_EVENT_WRITE_DONE_V2`，驱动收到后调用 `snd_pcm_period_elapsed()` |
| 为什么需要共享内存而不是直接传数据？ | 性能：避免大量数据在 CPU-ADSP 间拷贝；ADSP 直接 DMA 读取零拷贝，效率最高 |

### 12.9 与手机平台对比

手机平台（使用 SLIMBUS/I2S）的流程完全一致，只是 AFE port 不同：

| 项目 | Auto 平台 | 手机平台 |
|------|-----------|----------|
| ASM session | 相同 | 相同 |
| 共享内存机制 | 相同 | 相同 |
| APR 通信 | 相同 | 相同 |
| AFE port | TERT_TDM_RX_0/1/2 | SLIMBUS_0_RX |
| 物理接口 | TDM 总线 → 外部 DSP/Codec | SLIMBUS → WSA881x / 内部 Codec |

### 12.10 涉及的 APR 命令汇总

| APR 命令 | 方向 | 作用 | 代码位置 |
|----------|------|------|----------|
| `ASM_CMD_SHARED_MEM_MAP_REGIONS` (0x00010D92) | Kernel → ADSP | 映射共享内存到 ADSP 地址空间 | q6asm.c:386 |
| `ASM_CMDRSP_SHARED_MEM_MAP_REGIONS` (0x00010D93) | ADSP → Kernel | 映射结果（返回 mem_map_handle） | q6asm.c:404 |
| `ASM_CMD_SHARED_MEM_UNMAP_REGIONS` (0x00010D94) | Kernel → ADSP | 取消共享内存映射 | q6asm.c |
| `ASM_STREAM_CMD_OPEN_WRITE_V3` (0x00010DB3) | Kernel → ADSP | 打开 ASM 写会话 | q6asm.c |
| `ASM_DATA_CMD_WRITE_V2` (0x00010DAB) | Kernel → ADSP | 通知 ADSP 读取共享内存中的音频数据 | q6asm.c:1231 |
| `ASM_DATA_EVENT_WRITE_DONE_V2` (0x00010D99) | ADSP → Kernel | ADSP 通知一个 period 处理完毕 | q6asm.c:593 |
| `ASM_SESSION_CMD_RUN_V2` (0x00010DAA) | Kernel → ADSP | 启动 ASM session | q6asm.c |
| `AFE_PORT_CMD_DEVICE_START` (0x000100E5) | Kernel → ADSP | 启动 AFE port (TDM/I2S/SLIMBUS) | q6afe.c:1351 |
| `ADM_CMDRSP_DEVICE_OPEN_V5` | Kernel → ADSP | 打开 ADM COPP (路由 ASM→AFE) | q6adm.c |

### 12.11 关键函数索引

| 函数 | 文件 | 行号 | 作用 |
|------|------|------|------|
| `pcm_write()` | `vendor/qcom/opensource/tinyalsa/pcm.c` | 534 | 用户空间写入：ioctl→copy_from_user→DMA buffer |
| `snd_pcm_xferi_frames_ioctl()` | `kernel/.../sound/core/pcm_native.c` | 2842 | 内核 ioctl 处理入口 |
| `default_write_copy()` | `kernel/.../sound/core/pcm_lib.c` | 1926 | copy_from_user 到 DMA buffer |
| `q6asm_dai_open()` | `kernel/.../sound/soc/qcom/qdsp6/q6asm-dai.c` | 325 | 打开 PCM 子流，分配 audio client 和 DMA buffer |
| `q6asm_dai_prepare()` | `kernel/.../sound/soc/qcom/qdsp6/q6asm-dai.c` | 210 | prepare 阶段：映射共享内存 + open_write |
| `q6asm_map_memory_regions()` | `kernel/.../sound/soc/qcom/qdsp6/q6asm.c` | 423 | 将 DMA buffer 物理地址映射到 ADSP |
| `__q6asm_memory_map_regions()` | `kernel/.../sound/soc/qcom/qdsp6/q6asm.c` | 344 | 发送 ASM_CMD_SHARED_MEM_MAP_REGIONS APR 命令 |
| `q6asm_write_async()` | `kernel/.../sound/soc/qcom/qdsp6/q6asm.c` | 1205 | 发送 ASM_DATA_CMD_WRITE_V2 通知 ADSP 读数据 |
| `apr_send_pkt()` | `kernel/.../soc/qcom/apr.c` | - | APR 包发送接口 |
| `event_handler()` | `kernel/.../sound/soc/qcom/qdsp6/q6asm-dai.c` | 173 | ADSP 回调：处理 WRITE_DONE，触发下一个 write |
| `q6asm_callback()` | `kernel/.../sound/soc/qcom/qdsp6/q6asm.c` | 593 | ASM APR 回调：解析 WRITE_DONE_V2 |
| `q6afe_port_start()` | `kernel/.../sound/soc/qcom/qdsp6/q6afe.c` | 1306 | 启动 AFE port（TDM 输出通道） |
| `q6adm_open()` | `kernel/.../sound/soc/qcom/qdsp6/q6adm.c` | 383 | 打开 ADM COPP（ASM→AFE 路由） |
| `snd_pcm_period_elapsed()` | `kernel/.../sound/core/pcm_lib.c` | - | 通知 ALSA core 一个 period 已消费完 |

---

## 13. ADSP 通信架构：HAL 与驱动的关系

> 核心结论：**ADSP 只与内核驱动通信，不与 Audio HAL 直接通信**。HAL 通过标准 ALSA ioctl 和 audio_cal ioctl 间接操作，内核驱动通过 APR 子系统与 ADSP 交互。

### 13.1 两条通信路径总览

```
┌─────────────────────────────────────────────────────────────┐
│ Audio HAL (用户空间)                                         │
│                                                              │
│  路径一: PCM 数据            路径二: ACDB 校准               │
│  pcm_write()                 acdb_send_audio_cal_v4()        │
│  pcm_open()                       ↓                          │
│  mixer_ctl_set_value()       libacdbloader.so                │
│       │                        │                              │
│       │ ioctl()                │ acdb_ioctl() ← 本地函数      │
│       │ (ALSA PCM/mixer)      │ 读取 .acdb 文件数据          │
│       │                        │                              │
│       │                        ↓ ioctl()                      │
│       │                   AUDIO_SET_CALIBRATION               │
│       │                   (/dev/msm_audio_cal)                │
└───────┼────────────────────────┼─────────────────────────────┘
        │                        │
        │  ← 全部通过内核 →      │
        ▼                        ▼
┌─────────────────────────────────────────────────────────────┐
│ 内核空间                                                     │
│                                                              │
│  ALSA ASoC              Audio Cal Driver                     │
│  q6asm_dai.c            msm_audio_cal                        │
│       │                        │                              │
│       ▼                        ▼                              │
│  ┌─────────────────────────────────────┐                     │
│  │          APR 子系统                  │                     │
│  │  (Asynchronous Packet Router)       │                     │
│  │  CPU ↔ ADSP 通信的唯一通道          │                     │
│  └─────────────────┬───────────────────┘                     │
└────────────────────┼────────────────────────────────────────┘
                     │ APR (glancer/spmi 硬件链路)
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ ADSP (独立 DSP 处理器)                                       │
│  ASM / ADM / AFE / CVD                                      │
└─────────────────────────────────────────────────────────────┘
```

### 13.2 路径一：PCM 数据路径

**HAL → 内核 ALSA → q6asm_dai → APR → ADSP ASM**

| 阶段 | 接口 | 操作 |
|------|------|------|
| HAL → 内核 | `pcm_write()` → `ioctl(SNDRV_PCM_IOCTL_WRITEI_FRAMES)` | 音频数据 copy_from_user 到 DMA buffer |
| HAL → 内核 | `pcm_open()` → `ioctl(SNDRV_PCM_IOCTL_OPEN)` | 打开 PCM 设备 `/dev/snd/pcmCxDxp` |
| HAL → 内核 | `mixer_ctl_set_value()` → ALSA mixer ioctl | 设置 mixer 控制器（路由、音量等） |
| 内核 → ADSP | `q6asm_write_async()` → `apr_send_pkt(ASM_DATA_CMD_WRITE_V2)` | 通知 ADSP 从共享内存读数据 |
| 内核 → ADSP | `q6asm_map_memory_regions()` → `apr_send_pkt(ASM_CMD_SHARED_MEM_MAP_REGIONS)` | 映射共享内存到 ADSP |
| ADSP → 内核 | `ASM_DATA_EVENT_WRITE_DONE_V2` → APR 回调 | 通知内核数据已处理完 |
| 内核 → HAL | `snd_pcm_period_elapsed()` → poll/ioctl 返回 | 通知 HAL 可写下一个 period |

**HAL 侧代码**（`audio_hw.c`）：

```c
// HAL 只操作 ALSA 接口，不直接与 ADSP 通信
static ssize_t out_write(struct audio_stream_out *stream, const void *buffer, size_t bytes)
{
    // ...
    ret = pcm_write(out->pcm, (void *)buffer, bytes);  // ALSA PCM 写入
}
```

**内核侧代码**（`q6asm-dai.c` → `q6asm.c`）：

```c
// 内核驱动通过 APR 与 ADSP 通信
int q6asm_write_async(struct audio_client *ac, uint32_t len, ...)
{
    pkt->hdr.opcode = ASM_DATA_CMD_WRITE_V2;         // APR 命令
    write->buf_addr_lsw = lower_32_bits(ab->phys);    // 共享内存物理地址
    write->buf_addr_msw = upper_32_bits(ab->phys);
    write->mem_map_handle = ac->port[...].mem_map_handle;
    rc = apr_send_pkt(ac->adev, pkt);                 // 发送给 ADSP
}
```

### 13.3 路径二：ACDB 校准路径

**HAL → libacdbloader.so → 内核 cal driver → APR → ADSP**

#### 13.3.1 HAL 调用链

```c
// audio_hw.c: enable_audio_route() 中调用
audio_extn_utils_send_audio_calibration(adev, usecase);
  → platform_send_audio_calibration(platform, usecase, app_type);
    → acdb_dev_id = acdb_device_table[snd_device];   // 查 ACDB ID
    → my_data->acdb_send_audio_cal_v4(acdb_dev_id, ...);  // 调用 libacdbloader
```

**代码位置**：`platform.c:4989`

```c
int platform_send_audio_calibration(void *platform, struct audio_usecase *usecase,
                                    int app_type)
{
    // 查找 snd_device 对应的 ACDB ID
    acdb_dev_id = acdb_device_table[platform_get_spkr_prot_snd_device(snd_device)];

    // 调用 libacdbloader.so 的校准下发函数
    if (my_data->acdb_send_audio_cal_v4) {
        my_data->acdb_send_audio_cal_v4(acdb_dev_id, acdb_dev_type,
                                        app_type, sample_rate, i,
                                        backend_cfg.sample_rate);
    } else if (my_data->acdb_send_audio_cal_v3) {
        my_data->acdb_send_audio_cal_v3(acdb_dev_id, acdb_dev_type,
                                        app_type, sample_rate, i);
    } else if (my_data->acdb_send_audio_cal) {
        my_data->acdb_send_audio_cal(acdb_dev_id, acdb_dev_type, app_type,
                                     sample_rate);
    }
}
```

#### 13.3.2 libacdbloader.so 内部流程

libacdbloader.so 通过 `dlopen`/`dlsym` 动态加载（`platform.c:3290`）：

```c
my_data->acdb_handle = dlopen(LIB_ACDB_LOADER, RTLD_NOW);  // "libacdbloader.so"
my_data->acdb_send_audio_cal = dlsym(my_data->acdb_handle, "acdb_loader_send_audio_cal_v2");
my_data->acdb_send_audio_cal_v3 = dlsym(my_data->acdb_handle, "acdb_loader_send_audio_cal_v3");
my_data->acdb_send_audio_cal_v4 = dlsym(my_data->acdb_handle, "acdb_loader_send_audio_cal_v4");
```

libacdbloader 内部两步操作：

**Step 1**: `acdb_ioctl()` — 本地函数调用，从 `.acdb` 文件读取校准数据

```c
// acdb-loader.c
// acdb_ioctl 是纯本地函数，不涉及任何内核通信
// 它从 .acdb 二进制文件中查询校准数据（topology、audproc、afe 等）
result = acdb_ioctl(ACDB_CMD_GET_AUDPROC_COMMON_TABLE,
        (const uint8_t *)&audtable, sizeof(audtable),
        (uint8_t *)&response, sizeof(response));
```

> `acdb_ioctl` 定义在 `vendor/qcom/proprietary/mm-audio-cal/audcal/acdb/src/acdb.c`，是纯用户空间的文件查询操作，根据 ACDB ID 从 .acdb 文件中检索对应的校准数据块。

**Step 2**: `ioctl(AUDIO_SET_CALIBRATION)` — 将校准数据发送给内核

```c
// acdb-loader.c
// 通过 /dev/msm_audio_cal 设备节点将校准数据发送给内核 cal driver
result = ioctl(cal_driver_handle, AUDIO_SET_CALIBRATION, &audproc_cal);
```

#### 13.3.3 各种校准数据的下发

| 校准类型 | 函数 | ioctl 数据结构 | 内核处理 → ADSP 目标 |
|----------|------|----------------|---------------------|
| AFE topology | `send_afe_topology()` | `audio_cal_afe_top` | AFE port 配置 → ADSP AFE |
| ADM topology | `send_adm_topology()` | `audio_cal_adm_top` | ADM COPP → ADSP ADM |
| ASM topology | `send_asm_topology()` | `audio_cal_asm_top` | ASM → ADSP ASM |
| ASM audstrm | `send_audstrmtable()` | `audio_cal_audstrm` | ASM stream → ADSP ASM |
| ADM audproc | `send_audproc_cal()` | `audio_cal_audproc` | ADM COPP → ADSP ADM |
| ADM audvol | `send_audvoltable()` | `audio_cal_audvol` | ADM COPP → ADSP ADM |
| AFE cal | `send_afe_cal()` | `audio_cal_afe` | AFE → ADSP AFE |
| Common custom topo | `send_common_custom_topology()` | `audio_cal_basic` | Core → ADSP |
| AFE custom topo | `send_afe_custom_topology()` | `audio_cal_basic` | AFE → ADSP AFE |
| ADM custom topo | `send_adm_custom_topology()` | `audio_cal_basic` | ADM → ADSP ADM |
| ASM custom topo | `send_asm_custom_topology()` | `audio_cal_basic` | ASM → ADSP ASM |
| Meta info | `send_meta_info()` | `audio_cal_metainfo` | Core → ADSP |
| HW delay | `send_hw_delay()` | `audio_cal_hw_delay` | Core → ADSP |

所有函数最终都调用 `ioctl(cal_driver_handle, AUDIO_SET_CALIBRATION, &xxx_cal)`，**全部经过内核**。

### 13.4 内核 cal driver 到 ADSP 的传递

内核 `msm_audio_cal` 驱动收到 `AUDIO_SET_CALIBRATION` ioctl 后：

```
AUDIO_SET_CALIBRATION ioctl
  → 内核 cal driver 解析 cal_type
  → 根据校准类型，通过 APR 发送对应命令给 ADSP：
      AFE topology  → AFE port set_param → ADSP AFE
      ADM topology  → ADM device open    → ADSP ADM
      ASM topology  → ASM stream config  → ADSP ASM
      audproc cal   → ADM COPP set_param → ADSP ADM
      AFE cal       → AFE port set_param → ADSP AFE
      custom topo   → ASM/ADM/AFE 上传   → ADSP
```

### 13.5 HAL 操作的设备节点汇总

| 设备节点 | HAL 操作 | 内核驱动 | 作用 |
|----------|----------|----------|------|
| `/dev/snd/pcmCxDxp` | `pcm_open/read/write/ioctl` | `q6asm_dai` (ASoC) | PCM 数据读写 |
| `/dev/snd/controlCxD` | `mixer_open/get_ctl/set_value` | ALSA control | Mixer 控制（路由/音量） |
| `/dev/msm_audio_cal` | `ioctl(AUDIO_SET_CALIBRATION)` | `msm_audio_cal` | ACDB 校准数据下发 |
| `/dev/snd/timer` | `timer_open/read` | ALSA timer | 音频定时器（低延迟同步） |

### 13.6 为什么必须经过内核？

| 原因 | 说明 |
|------|------|
| **安全隔离** | ADSP 运行在独立 DSP 处理器上，用户空间（HAL）没有权限直接访问 ADSP 地址空间。APR 是内核子系统，是 CPU ↔ ADSP 通信的唯一合法通道 |
| **共享内存管理** | DMA buffer 的物理连续内存分配、页表映射、内存映射到 ADSP 地址空间，这些都需要内核权限 |
| **硬件资源管控** | PCM 设备、AFE port、ADM COPP 等硬件资源由内核驱动统一管理，避免用户空间冲突 |
| **并发安全** | 多个 HAL stream 可能同时操作，内核负责互斥和资源调度 |
| **SELinux 策略** | Android SELinux 限制用户空间进程只能访问特定设备节点，无法直接操作硬件 |

### 13.7 完整通信链路对比表

| 通信方向 | HAL 操作 | 用户空间→内核接口 | 内核→ADSP 接口 | ADSP 模块 |
|----------|----------|-------------------|----------------|-----------|
| 音频数据下行 | `pcm_write()` | ALSA PCM ioctl | `ASM_DATA_CMD_WRITE_V2` (APR) | ASM |
| 音频数据上行 | `pcm_read()` | ALSA PCM ioctl | `ASM_DATA_CMD_READ_V2` (APR) | ASM |
| 设备路由 | `mixer_ctl_set_value()` | ALSA mixer ioctl | ALSA ASoC → `AFE_PORT_CMD_DEVICE_START` (APR) | AFE |
| 音量设置 | `out_set_volume()` | ALSA PCM ioctl | `ASM_DATA_CMD_SET_VOL_V2` (APR) | ASM |
| AFE 校准 | `acdb_send_audio_cal()` | `AUDIO_SET_CALIBRATION` (cal ioctl) | `AFE_PORT_CMD_SET_PARAM_V2` (APR) | AFE |
| ADM 校准 | `acdb_send_audio_cal()` | `AUDIO_SET_CALIBRATION` (cal ioctl) | `ADM_CMD_DEVICE_OPEN_V5` (APR) | ADM |
| ASM 校准 | `acdb_send_audio_cal()` | `AUDIO_SET_CALIBRATION` (cal ioctl) | `ASM_STREAM_CMD_SET_ENCDEC_PARAM` (APR) | ASM |
| 内存映射 | (自动，prepare 阶段) | ALSA PCM ioctl (trigger) | `ASM_CMD_SHARED_MEM_MAP_REGIONS` (APR) | ASM |

### 13.8 关键函数索引

| 函数 | 文件 | 作用 |
|------|------|------|
| `platform_send_audio_calibration()` | `platform.c:4989` | HAL 入口：查 ACDB ID，调用 libacdbloader |
| `acdb_ioctl()` | `acdb.c:1344` | 纯本地函数：从 .acdb 文件查询校准数据 |
| `send_afe_topology()` | `acdb-loader.c` | 下发 AFE topology → `ioctl(AUDIO_SET_CALIBRATION)` |
| `send_adm_topology()` | `acdb-loader.c` | 下发 ADM topology → `ioctl(AUDIO_SET_CALIBRATION)` |
| `send_asm_topology()` | `acdb-loader.c` | 下发 ASM topology → `ioctl(AUDIO_SET_CALIBRATION)` |
| `send_audproc_cal()` | `acdb-loader.c` | 下发 audproc 校准 → `ioctl(AUDIO_SET_CALIBRATION)` |
| `send_afe_cal()` | `acdb-loader.c` | 下发 AFE 校准 → `ioctl(AUDIO_SET_CALIBRATION)` |
| `q6asm_write_async()` | `q6asm.c:1205` | 内核通过 APR 通知 ADSP 读数据 |
| `q6asm_map_memory_regions()` | `q6asm.c:423` | 内核通过 APR 映射共享内存到 ADSP |
| `q6afe_port_start()` | `q6afe.c:1306` | 内核通过 APR 启动 AFE port |
| `q6adm_open()` | `q6adm.c:383` | 内核通过 APR 打开 ADM COPP |
| `apr_send_pkt()` | `kernel/.../soc/qcom/apr.c` | APR 子系统：CPU ↔ ADSP 通信核心接口 |

---

## 14. TDM Slot 对应关系

> 梳理 Auto 平台 TDM（Time Division Multiplexing）总线中，每个 slot 如何与 Car Audio Bus、usecase、AFE port 对应。核心数据来自内核机器驱动 `sa8155.c` 中的 `tdm_slot[]`、`tdm_rx_cfg[]`、`tdm_rx_slot_offset[]` 三张表。

### 14.1 TDM Slot 基础配置

**代码位置**：`kernel/msm-5.4/techpack/audio/asoc/sa8155.c:476`

```c
struct tdm_slot_cfg {
    u32 width;   // 每个 slot 的位宽（bit）
    u32 num;     // 每帧的 slot 数量
};

static struct tdm_slot_cfg tdm_slot[TDM_INTERFACE_MAX] = {
    /* PRI TDM */  {16, 16},   // slot_width=16bit, num=16 slots
    /* SEC TDM */  {32, 8},    // slot_width=32bit, num=8 slots
    /* TERT TDM */ {32, 8},    // slot_width=32bit, num=8 slots
    /* QUAT TDM */ {32, 8},    // slot_width=32bit, num=8 slots
    /* QUIN TDM */ {32, 16},   // slot_width=32bit, num=16 slots
};
```

| TDM 接口 | 枚举值 | slot_width | num_slots | 帧大小 (bit) | 用途 |
|----------|--------|------------|-----------|-------------|------|
| PRI TDM | TDM_PRI=0 | 16 | 16 | 256 | 预留 |
| SEC TDM | TDM_SEC=1 | 32 | 8 | 256 | 预留 |
| TERT TDM | TDM_TERT=2 | 32 | 8 | 256 | Primary Zone 音频输出 |
| QUAT TDM | TDM_QUAT=3 | 32 | 8 | 256 | Front Passenger / 辅助输出 |
| QUIN TDM | TDM_QUIN=4 | 32 | 16 | 512 | Rear Seat 16通道输出 |

### 14.2 TDM RX 通道数配置

**代码位置**：`sa8155.c:191`

```c
struct dev_config {
    u32 sample_rate;
    u32 bit_format;
    u32 channels;
};

static struct dev_config tdm_rx_cfg[TDM_INTERFACE_MAX][TDM_PORT_MAX] = {
    ...
    { /* TERT TDM */
        {48KHz, S16_LE, 6},  /* RX_0: 6 channels (media + sys_notification) */
        {48KHz, S16_LE, 1},  /* RX_1: 1 channel  (nav_guidance) */
        {48KHz, S16_LE, 1},  /* RX_2: 1 channel  (phone) */
        {48KHz, S16_LE, 1},  /* RX_3 */
        {48KHz, S16_LE, 1},  /* RX_4 */
        {48KHz, S16_LE, 1},  /* RX_5 */
        {48KHz, S16_LE, 1},  /* RX_6 */
        {48KHz, S16_LE, 1},  /* RX_7 */
    },
    { /* QUAT TDM */
        {48KHz, S24_LE, 1},  /* RX_0: nav */
        {48KHz, S24_LE, 1},  /* RX_1: vr downstream */
        {48KHz, S24_LE, 1},  /* RX_2: phone */
        {48KHz, S24_LE, 2},  /* RX_3: asc */
        {48KHz, S24_LE, 1},  /* RX_4: ring */
        {48KHz, S24_LE, 1},  /* RX_5: animation */
        ...
    },
    { /* QUIN TDM */
        {48KHz, S16_LE, 16}, /* RX_0: 16 channels (rear seat 16CH SPKR) */
        ...
    },
};
```

### 14.3 TDM RX Slot Offset 映射（核心）

**代码位置**：`sa8155.c:519`

offset 值表示该 port 的音频数据在 TDM 帧中的**起始字节偏移**，`0xFFFF` 表示结束/未使用。

#### TERT TDM RX

```c
{/* TERT TDM */
    {0, 4, 8, 12, 16, 20, 0xFFFF},  /* RX_0: 6ch, offset 0~20 */
    {24, 0xFFFF},                     /* RX_1: 1ch, offset 24   */
    {28, 0xFFFF},                     /* RX_2: 1ch, offset 28   */
    {0xFFFF},  /* RX_3 not used */
    {0xFFFF},  /* RX_4 not used */
    {0xFFFF},  /* RX_5 not used */
    {0xFFFF},  /* RX_6 not used */
    {0xFFFF},  /* RX_7 not used */
},
```

#### QUAT TDM RX

```c
{/* QUAT TDM */
    {0, 0xFFFF},           /* RX_0: 1ch, offset 0  — nav */
    {4, 0xFFFF},           /* RX_1: 1ch, offset 4  — vr downstream */
    {8, 0xFFFF},           /* RX_2: 1ch, offset 8  — phone */
    {24, 28, 0xFFFF},      /* RX_3: 2ch, offset 24,28 — asc */
    {16, 0xFFFF},          /* RX_4: 1ch, offset 16 — ring */
    {12, 0xFFFF},          /* RX_5: 1ch, offset 12 — animation */
    {0xFFFF},              /* RX_6 not used */
    {0xFFFF},              /* RX_7 not used */
    {60, 0xFFFF},          /* extra */
},
```

#### QUIN TDM RX

```c
{/* QUIN TDM */
    {0, 4, 8, 12, 16, 20, 24, 28,
     32, 36, 40, 44, 48, 52, 56, 60, 0xFFFF},  /* RX_0: 16ch, offset 0~60 */
    {0xFFFF},  /* RX_1 not used */
    {0xFFFF},  /* RX_2 not used */
    {0xFFFF},  /* RX_3 not used */
    {0xFFFF},  /* RX_4 not used */
    {0xFFFF},  /* RX_5 not used */
    {0xFFFF},  /* RX_6 not used */
    {60, 0xFFFF},  /* extra */
},
```

### 14.4 TERT TDM 帧结构图

TERT TDM：8 slots × 32bit/slot = 256bit/frame，采样率 48kHz

```
     Slot 0    Slot 1    Slot 2    Slot 3    Slot 4    Slot 5    Slot 6    Slot 7
    ┌─────────┬─────────┬─────────┬─────────┬─────────┬─────────┬─────────┬─────────┐
    │ Byte    │ Byte    │ Byte    │ Byte    │ Byte    │ Byte    │ Byte    │ Byte    │
    │  0~3    │  4~7    │  8~11   │ 12~15   │ 16~19   │ 20~23   │ 24~27   │ 28~31   │
    │         │         │         │         │         │         │         │         │
    │ MEDIA L │ MEDIA R │  SYS L  │  SYS R  │ NOTIF L │ NOTIF R │   NAV   │  PHONE  │
    │         │         │         │         │         │         │         │         │
    ├─────────┴─────────┴─────────┴─────────┴─────────┴─────────┼─────────┬─────────┤
    │           TERT_TDM_RX_0  (6 channels)                     │RX_1(1ch)│RX_2(1ch)│
    └────────────────────────────────────────────────────────────┴─────────┴─────────┘

    ←———————— TERT_TDM_RX_0: MEDIA + SYS_NOTIFICATION ————————→ ←— NAV —→ ←PHONE→
```

### 14.5 QUAT TDM 帧结构图

QUAT TDM：8 slots × 32bit/slot = 256bit/frame，采样率 48kHz，S24_LE

```
     Slot 0    Slot 1    Slot 2    Slot 3    Slot 4    Slot 5    Slot 6    Slot 7
    ┌─────────┬─────────┬─────────┬─────────┬─────────┬─────────┬─────────┬─────────┐
    │ Byte    │ Byte    │ Byte    │ Byte    │ Byte    │ Byte    │ Byte    │ Byte    │
    │  0~3    │  4~7    │  8~11   │ 12~15   │ 16~19   │ 20~23   │ 24~27   │ 28~31   │
    │         │         │         │         │         │         │         │         │
    │   NAV   │VR_DOWN  │  PHONE  │ANIMATN  │  RING   │  (rsv)  │ ASC L   │ ASC R   │
    │         │ STREAM  │         │         │         │         │         │         │
    ├─────────┬─────────┬─────────┬─────────┬─────────┬─────────┼─────────┴─────────┤
    │RX_0(1ch)│RX_1(1ch)│RX_2(1ch)│RX_5(1ch)│RX_4(1ch)│ (rsv)  │  RX_3 (2ch)       │
    └─────────┴─────────┴─────────┴─────────┴─────────┴─────────┴───────────────────┘
```

### 14.6 QUIN TDM 帧结构图

QUIN TDM：16 slots × 32bit/slot = 512bit/frame，采样率 48kHz，S16_LE

```
     Slot 0    Slot 1   ...  Slot 7    Slot 8   ...  Slot 14   Slot 15
    ┌─────────┬─────────┬───┬─────────┬─────────┬───┬─────────┬─────────┐
    │ Byte    │ Byte    │   │ Byte    │ Byte    │   │ Byte    │ Byte    │
    │  0~3    │  4~7    │   │ 28~31   │ 32~35   │   │ 56~59   │ 60~63   │
    │  CH0    │  CH1    │...│  CH7    │  CH8    │...│  CH14   │  CH15   │
    ├─────────┴─────────┴───┴─────────┴─────────┴───┴─────────┴─────────┤
    │              QUIN_TDM_RX_0  (16 channels)                         │
    │              Rear Seat Zone 全部音频输出                            │
    └────────────────────────────────────────────────────────────────────┘
```

### 14.7 完整 Bus → TDM → Slot 对应总表

| Car Audio Bus | Usecase | FE PCM | BE TDM Port | TDM 通道数 | Slot 偏移(字节) | mixer path |
|---|---|---|---|---|---|---|
| BUS00_MEDIA | PLAYBACK_MEDIA | MultiMedia1 (pcm0) | TERT_TDM_RX_0 | 6 | 0,4,8,12,16,20 | media-playback |
| BUS01_SYS_NOTIFICATION | PLAYBACK_SYS_NOTIFICATION | MultiMedia5 (pcm9) | TERT_TDM_RX_0 | (同RX_0) | (同RX_0) | sys-notification-playback |
| BUS02_NAV_GUIDANCE | PLAYBACK_NAV_GUIDANCE | MultiMedia2 (pcm1) | TERT_TDM_RX_1 | 1 | 24 | nav-guidance-playback |
| BUS03_PHONE | PLAYBACK_PHONE | MultiMedia10 (pcm12) | TERT_TDM_RX_2 | 1 | 28 | phone-playback |
| BUS08_FRONT_PASSENGER | PLAYBACK_FRONT_PASSENGER | MultiMedia23 (pcm55) | QUAT_TDM_RX_0 | 8 | 0~28 | front-passenger-playback |
| BUS16_REAR_SEAT | PLAYBACK_REAR_SEAT | MultiMedia22 (pcm54) | QUIN_TDM_RX_0 | 16 | 0~60 | rear-seat-playback |

> **注意**：MEDIA 和 SYS_NOTIFICATION 共享 `TERT_TDM_RX_0` 的 6 个 slot。ADSP 内部通过 ASM session routing 将两个 FE PCM 流混音后输出到同一个 BE port 的不同 slot 位置。

### 14.8 MEDIA 与 SYS_NOTIFICATION 共享 RX_0 的原理

```
┌──────────────────────────────────────────────────────┐
│ ADSP ASM                                              │
│                                                       │
│  Session A (MultiMedia1):                              │
│    MEDIA PCM data → ASM 解码/音效 → 混音 → Slot 0,1  │
│                                                       │
│  Session B (MultiMedia5):                              │
│    SYS_NOTIF PCM data → ASM 解码/音效 → 混音 → Slot 2,3,4,5 │
│                                                       │
│  混音后输出到 AFE TERT_TDM_RX_0:                       │
│    Slot 0,1 = MEDIA (L,R)                             │
│    Slot 2,3 = SYS (L,R)                               │
│    Slot 4,5 = NOTIFICATION (L,R)                      │
└──────────────────────────────────────────────────────┘
```

两个 ASM session 通过 ADM COPP 路由到同一个 AFE port，ADSP ADM 负责将两路音频按 slot 位置合路到同一个 TDM 帧。

### 14.9 Slot Offset 如何下发到 ADSP

机器驱动 [sa8155.c](file:///home/kongchaochao/work/AI_25/apps/LINUX/android/kernel/msm-5.4/techpack/audio/asoc/sa8155.c) 在初始化和 PCM 打开时，通过 AFE port 配置将 slot 信息发送给 ADSP：

```c
// sa8155.c: 构造 AFE TDM 配置
tdm_port.tdm.nslots_per_frame = tdm_slot[intf_idx].num;   // 8 for TERT
tdm_port.tdm.slot_width = tdm_slot[intf_idx].width;        // 32 for TERT
tdm_port.tdm.num_channels = tdm_rx_cfg[intf_idx][port_idx].channels;
tdm_port.tdm.sample_rate = tdm_rx_cfg[intf_idx][port_idx].sample_rate;
tdm_port.tdm.bit_width = tdm_slot[intf_idx].width;

// Slot Mapping — 告诉 ADSP 每个通道在 TDM 帧中的字节偏移
for (i = 0; i < tdm_slot[intf_idx].num; i++)
    tdm_port.tdm.slot_mask |= 1 << i;  // 使能所有 slot

// 最终通过 AFE_PORT_CMD_SET_PARAM_V2 发送给 ADSP AFE
```

ADSP AFE 收到配置后，知道：
- **TERT_TDM_RX_0** 的 6 个通道从帧偏移 0, 4, 8, 12, 16, 20 读取
- **TERT_TDM_RX_1** 的 1 个通道从帧偏移 24 读取
- **TERT_TDM_RX_2** 的 1 个通道从帧偏移 28 读取

### 14.10 mixer_paths_adp.xml 中的 TDM 通道数设置

```xml
<!-- media-playback: 设置 TERT_TDM_RX_0 为 6 通道，路由 MultiMedia1 -->
<path name="media-playback">
    <ctl name="TERT_TDM_RX_0 Channels" value="Six" />
    <ctl name="TERT_TDM_RX_0 Audio Mixer MultiMedia1" value="1" />
</path>

<!-- sys-notification: 同样设置 TERT_TDM_RX_0 为 6 通道，路由 MultiMedia5 -->
<path name="sys-notification-playback">
    <ctl name="TERT_TDM_RX_0 Channels" value="Six" />
    <ctl name="TERT_TDM_RX_0 Audio Mixer MultiMedia5" value="1" />
</path>

<!-- nav-guidance: 设置 TERT_TDM_RX_1 为 1 通道，路由 MultiMedia2 -->
<path name="nav-guidance-playback">
    <ctl name="TERT_TDM_RX_1 Channels" value="One" />
    <ctl name="TERT_TDM_RX_1 Audio Mixer MultiMedia2" value="1" />
</path>

<!-- phone: 设置 TERT_TDM_RX_2 为 1 通道，路由 MultiMedia10 -->
<path name="phone-playback">
    <ctl name="TERT_TDM_RX_2 Channels" value="One" />
    <ctl name="TERT_TDM_RX_2 Audio Mixer MultiMedia10" value="1" />
</path>

<!-- front-passenger: 设置 QUAT_TDM_RX_0 为 8 通道，路由 MultiMedia23 -->
<path name="front-passenger-playback">
    <ctl name="QUAT_TDM_RX_0 Channels" value="Eight" />
    <ctl name="QUAT_TDM_RX_0 Audio Mixer MultiMedia23" value="1" />
</path>

<!-- rear-seat: 设置 QUIN_TDM_RX_0 为 16 通道，路由 MultiMedia22 -->
<path name="rear-seat-playback">
    <ctl name="QUIN_TDM_RX_0 Channels" value="Sixteen" />
    <ctl name="QUIN_TDM_RX_0 Audio Mixer MultiMedia22" value="1" />
</path>
```

`XXX Channels` mixer 控制对应 `tdm_rx_cfg[port.mode][port.channel].channels`，设置 ALSA ASoC 驱动中的 TDM 通道数，最终通过 AFE port 配置传递给 ADSP。

### 14.11 自定义 Slot Offset 表

代码中还有一个 `tdm_rx_slot_offset_custom[]` 表（`sa8155.c:737`），初始全为 `0xFFFF`（未使用），供 OEM 厂商根据实际硬件接线覆盖默认 slot 分配：

```c
/*****************************************************************************
 * TO BE UPDATED: Codec/Platform specific tdm slot offset table
 * NOTE:
 *     Each entry represents the slot offset array of one backend tdm device
 *     valid offset represents the starting offset in byte for the channel
 *     use 0xFFFF for end or unused slot offset entry.
 *****************************************************************************/
static unsigned int tdm_rx_slot_offset_custom
    [TDM_INTERFACE_MAX][TDM_PORT_MAX][TDM_SLOT_OFFSET_MAX] = {
    {/* PRI TDM */
        {0xFFFF}, /* not used */
        ...
    },
    ...
};
```

OEM 可通过设备树 (DT) 属性覆盖默认配置，或直接修改 `tdm_rx_slot_offset_custom[]` 表来适配不同的外部 DSP/Codec 接线方案。

### 14.12 关键数据结构与函数索引

| 数据结构/函数 | 文件 | 行号 | 作用 |
|---|---|---|---|
| `tdm_slot[]` | `sa8155.c` | 476 | TDM 接口的 slot_width 和 num_slots 配置 |
| `tdm_rx_cfg[]` | `sa8155.c` | 191 | TDM RX port 的采样率、位宽、通道数 |
| `tdm_rx_slot_offset[]` | `sa8155.c` | 519 | TDM RX port 在帧中的字节偏移（核心映射表） |
| `tdm_rx_slot_offset_custom[]` | `sa8155.c` | 737 | OEM 自定义 slot offset 覆盖表 |
| `tdm_slot_custom[]` | `sa8155.c` | 502 | OEM 自定义 slot 配置覆盖表 |
| `TDM_INTERFACE_MAX` 枚举 | `sa8155.c` | 167 | TDM 接口枚举：PRI/SEC/TERT/QUAT/QUIN/SEN/SEP/HSIF0/1/2 |
| AFE TDM 配置发送 | `sa8155.c` | 2576 | `AFE_PORT_CMD_SET_PARAM_V2` → ADSP AFE |
| Slot mapping 配置 | `sa8155.c` | 2604 | `slot_mapping_v2.offset[]` → ADSP AFE |
| `TERT_TDM_RX_0 Channels` | `mixer_paths_adp.xml` | 1301 | mixer 控制设置 TDM 通道数 |

---

## 15. HAL platform.c 与内核 platform_driver 的关系

> 澄清 Audio HAL 中 `platform.c` 文件与 Linux 内核 `platform_driver` 之间的命名混淆——两者虽然都叫 "platform"，但含义完全不同，没有代码层面的直接关联。

### 15.1 命名对比

| | Audio HAL `platform.c` | 内核 `platform_driver` / `platform_device` |
|---|---|---|
| **所在层** | 用户空间 (HAL) | 内核空间 |
| **"platform" 含义** | **芯片平台/硬件平台**（如 msm8974、msmnile） | **Linux 设备驱动模型**（platform bus） |
| **核心内容** | 平台相关的音频配置：snd_device 表、ACDB ID 映射、backend 名称、XML 路径等 | 通过 `platform_driver_register()` 注册到 platform bus 的驱动 |
| **数据结构** | `struct platform_data`（HAL 自定义，存 fluence/hd_voice 等配置） | `struct platform_driver`（Linux 内核标准结构） |
| **文件命名原因** | 按 SoC 平台命名：`msm8974/platform.c` = "msm8974 平台相关的音频配置" | Linux 设备模型术语：platform = 总线类型之一 |

### 15.2 HAL platform.c 的职责

`platform.c` 是一个"平台配置大杂烩"文件，包含：

| 数据结构 | 作用 |
|----------|------|
| `device_table[]` | snd_device → mixer device name 映射 |
| `acdb_device_table[]` | snd_device → ACDB ID 映射 |
| `backend_tag_table[]` | snd_device → BE 后缀 |
| `hw_interface_table[]` | snd_device → HW interface |
| `use_case_table[]` | usecase → FE path name |
| `set_platform_defaults()` | 初始化上述各表默认值 |
| `platform_add_backend_name()` | 为 mixer path 追加 BE 后缀 |
| `platform_send_audio_calibration()` | 查 ACDB ID 并下发校准 |
| XML 解析逻辑 | 解析 `audio_platform_info.xml` 覆盖默认表 |

核心数据结构：

```c
// platform.c:304 — HAL 自定义的 platform_data
struct platform_data {
    struct audio_device *adev;
    bool fluence_in_spkr_mode;
    bool fluence_in_voice_call;
    bool hd_voice;
    bool is_wsa_speaker;
    // ... 全是音频特性开关
};
```

这里的 "platform" 就是**"芯片平台"**的意思——同一个 HAL 代码库要适配不同 SoC（msm8974、msmnile 等），不同平台的配置表不同，所以按平台拆分文件。

### 15.3 内核 platform_driver 的职责

Linux 内核的 `platform_driver` 是设备驱动模型概念，代表挂载在 platform bus 上的驱动：

```c
// 内核标准结构
struct platform_driver {
    int (*probe)(struct platform_device *);
    int (*remove)(struct platform_device *);
    struct device_driver driver;
    const struct platform_device_id *id_table;
};
```

在音频子系统中，内核 `platform_driver` 负责：
- 匹配设备树节点（`compatible` 匹配）
- 初始化硬件时钟（LPASS clock）、GPIO、DMA
- 注册 ASoC DAI driver / component driver（如 q6asm-dai、q6afe-dai）
- 注册 sound card（如 sa8155.c 中的 `snd_soc_register_card`）

### 15.4 两者的间接关联

```
┌─────────────────────────────────┐     ALSA ioctl      ┌──────────────────────────────────┐
│  Audio HAL (用户空间)            │ ──────────────────→ │  内核 ASoC (内核空间)              │
│                                 │                      │                                  │
│  platform.c:                    │  pcm_open()          │  platform_driver (sa8155.c):     │
│   - 查 device_table 决定开什么设备│  ────────────────→  │   - probe() 初始化硬件            │
│   - 查 use_case_table 拼 path   │  mixer_ctl_set()     │   - 注册 DAI / sound card        │
│   - 调 audio_route_apply_path() │  ────────────────→  │   - 创建 /dev/snd/pcmC0D0p       │
│   - 通过 tinyalsa 操作 ALSA     │                      │   - 创建 /dev/snd/controlC0      │
│                                 │                      │                                  │
│  struct platform_data {         │                      │  struct platform_driver {        │
│   fluence, hd_voice, ...        │                      │   probe, remove, ...             │
│  }                              │                      │  }                               │
└─────────────────────────────────┘                      └──────────────────────────────────┘
       ↑ 完全独立的数据结构                                  ↑ 完全独立的数据结构
```

两者通过 ALSA/ASoC 框架间接关联：
1. HAL 的 `platform.c` 通过 tinyalsa 的 `pcm_open()` / `mixer_ctl_set()` 操作内核的 ALSA 设备节点
2. 内核的 `platform_driver` 负责创建这些 ALSA 设备节点
3. **两者之间没有任何直接的函数调用或数据结构共享**

### 15.5 为什么都叫 "platform"

| 层级 | "platform" 的含义 | 举例 |
|------|-------------------|------|
| HAL | SoC 芯片平台代号 | `msm8974/platform.c` = msm8974 这个 SoC 的音频配置 |
| HAL XML | 同上 | `audio_platform_info.xml` = 平台信息配置文件 |
| 内核 ASoC | Linux platform bus 驱动 | `sa8155.c` 注册 `platform_driver` 到 platform bus |
| 内核 DAI | 同上 | `q6asm-dai.c` 通过 `module_platform_driver()` 注册 |

这是两个不同领域恰好使用了相同的英文单词：
- HAL 领域：platform = **硬件平台**（哪种 SoC）
- 内核领域：platform = **总线类型**（platform bus，与 USB bus、I2C bus 并列）

### 15.6 关键文件索引

| 文件 | 层级 | "platform" 含义 |
|------|------|-----------------|
| `vendor/.../hal/msm8974/platform.c` | HAL | SoC 平台配置 |
| `vendor/.../hal/msm8974/platform.h` | HAL | SoC 平台配置头文件 |
| `kernel/.../asoc/sa8155.c` (module_platform_driver) | 内核 | Linux platform bus 驱动 |
| `kernel/.../qdsp6/q6asm-dai.c` (module_platform_driver) | 内核 | Linux platform bus 驱动 |
| `kernel/.../qdsp6/q6afe-dai.c` (module_platform_driver) | 内核 | Linux platform bus 驱动 |
| `vendor/.../configs/msmnile_au/audio_platform_info.xml` | HAL XML | SoC 平台信息配置 |
