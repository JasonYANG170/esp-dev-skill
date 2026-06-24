# TinyUSB 示例索引

> 所有路径为真实仓库目录（相对 `D:/esp-skill/espressif-repos/tinyusb/`）。每个示例均含 `src/main.c`、`src/tusb_config.h`，设备侧另含 `src/usb_descriptors.c`。

## 设备示例（examples/device/）

| 路径 | 说明 |
|---|---|
| `examples/device/cdc_msc/` | CDC 虚拟串口 + MSC U 盘（RAM 盘），入门首选 |
| `examples/device/cdc_msc_freertos/` | 同上，FreeRTOS 集成版 |
| `examples/device/cdc_dual_ports/` | 双 CDC 实例 |
| `examples/device/cdc_uac2/` | CDC + UAC2 音频组合 |
| `examples/device/msc_dual_lun/` | MSC 双 LUN（两个逻辑盘） |
| `examples/device/hid_composite/` | HID 多 report（键盘/鼠标/消费者/手柄/触控笔） |
| `examples/device/hid_composite_freertos/` | 同上，FreeRTOS 版 |
| `examples/device/hid_boot_interface/` | HID boot 接口（键盘/鼠标） |
| `examples/device/hid_generic_inout/` | HID 通用 IN/OUT report |
| `examples/device/hid_multiple_interface/` | 多个 HID 接口 |
| `examples/device/webusb_serial/` | WebUSB + Vendor 类串口 |
| `examples/device/dfu/` | DFU 模式：download/upload/manifest 回调、多 alt 分区（固件升级，WIP） |
| `examples/device/dfu_runtime/` | DFU Runtime：detach 回调，应用运行时声明可进 DFU |
| `examples/device/midi_test/` | MIDI 设备：Note On/Off 旋律发送（stream API） |
| `examples/device/midi_test_freertos/` | MIDI + FreeRTOS |
| `examples/device/audio_test/` | UAC2 音频测试 |
| `examples/device/audio_test_freertos/` | UAC2 音频 + FreeRTOS |
| `examples/device/audio_test_multi_rate/` | UAC2 多采样率（Clock Unit RANGE 协商） |
| `examples/device/audio_4_channel_mic/` | UAC2 4 通道麦克风（IN 端点多通道） |
| `examples/device/audio_4_channel_mic_freertos/` | 同上 + FreeRTOS |
| `examples/device/uac2_headset/` | UAC2 耳机（扬声器+麦克风，双向，含音量控制） |
| `examples/device/uac2_speaker_fb/` | UAC2 喇叭 + 异步反馈端点（FIFO_COUNT 法，UAC1/UAC2 双协议） |
| `examples/device/mtp/` | MTP/PTP 媒体传输 |
| `examples/device/usbtmc/` | USBTMC 测试测量类（USB488，含 `visaQuery.py` 主机 SCPI 测试） |
| `examples/device/video_capture/` | UVC 视频采集（WIP） |
| `examples/device/video_capture_2ch/` | UVC 双通道采集 |
| `examples/device/net_lwip_webserver/` | RNDIS/ECM/NCM 网络 + lwIP Web 服务器 |
| `examples/device/dynamic_configuration/` | 动态切换 USB 配置（多配置描述符） |
| `examples/device/board_test/` | 板级自测 |

## 主机示例（examples/host/）

| 路径 | 说明 |
|---|---|
| `examples/host/cdc_msc_hid/` | 主机：枚举 CDC/MSC/HID 设备（入门首选） |
| `examples/host/cdc_msc_hid_freertos/` | 同上，FreeRTOS 版 |
| `examples/host/msc_file_explorer/` | 主机：U 盘文件浏览（FatFs） |
| `examples/host/hid_controller/` | 主机：读 HID 手柄/控制器 |
| `examples/host/midi_rx/` | 主机：接收 MIDI |
| `examples/host/device_info/` | 主机：打印设备描述符信息 |
| `examples/host/bare_api/` | 主机：底层裸 API 演示 |

## 双角色（OTG）示例（examples/dual/）

| 路径 | 说明 |
|---|---|
| `examples/dual/host_hid_to_device_cdc/` | 主机收 HID → 设备转发为 CDC |
| `examples/dual/host_info_to_device_cdc/` | 主机设备信息 → 设备 CDC 输出 |

## Type-C / Power Delivery（examples/typec/）

| 路径 | 说明 |
|---|---|
| `examples/typec/power_delivery/` | USB PD 3.0 sink：解析 Source Cap（PDO）、发 Request（RDO）、Accept/PS_READY（WIP，仅 STM32 G4，`only.txt` 限定） |

## 单元/硬件测试（test/）

| 路径 | 说明 |
|---|---|
| `test/unit-test/` | 单元测试（Ceedling）：`ceedling test:all` |
| `test/fuzz/` | Fuzz 测试 |
| `test/hil/` | 硬件在环测试 |

## 示例通用结构

```
examples/<device|host|dual>/<name>/
├── src/
│   ├── main.c              # 入口与主循环
│   ├── tusb_config.h       # 栈配置
│   ├── usb_descriptors.c   # 设备侧描述符回调（设备示例）
│   └── <class>.c           # 类后端/应用任务（如 msc_disk.c、hid_app.c）
├── CMakeLists.txt
├── CMakePresets.json
├── Makefile
└── skip.txt / prj.conf      # 测试/CI 配置
```

> `*_freertos` 后缀示例展示 RTOS 集成模式；音频示例多依赖 `lib/` 下的 lwIP/FreeRTOS（由 `get_deps.py` 管理）。
