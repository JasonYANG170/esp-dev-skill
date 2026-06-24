# Changelog

本文件记录 tinyusb-skill 的版本变更。格式遵循 [Keep a Changelog](https://keepachangelog.com/)。

## [1.1.0] - 2026-06-18

### Added
- 新增 5 个设备类/子系统场景配方，填补 README 一等公民支持但 recipe 缺失的设备类（Audio/MIDI/DFU/USBTMC/Type-C）：
  - `recipes/audio_uac2_device.md` — UAC2 音频设备（麦克风 IN/扬声器 OUT/耳机/异步反馈端点/采样率协商/音量控制）。来源示例：`examples/device/uac2_headset/`、`uac2_speaker_fb/`、`audio_4_channel_mic/`、`audio_test_multi_rate/`、`cdc_uac2/`。API：`tud_audio_write/read`、`tud_audio_feedback_params_cb`、`tud_audio_n_fb_set`、`TUD_AUDIO_EP_SIZE`。
  - `recipes/dfu_device.md` — DFU 固件升级（DFU 模式 vs DFU Runtime 区分、download/upload/manifest 回调、多分区 alt、bwPollTimeout、`tud_dfu_finish_flashing`）。来源示例：`examples/device/dfu/`、`dfu_runtime/`。API：`dfu_device.h` / `dfu_rt_device.h` / `dfu.h`。
  - `recipes/midi_device.md` — MIDI 设备（USB-MIDI 4 字节事件包、cable number/CIN、Note On/Off、packet vs stream API）。来源示例：`examples/device/midi_test/`、`midi_test_freertos/`、`host/midi_rx/`。API：`tud_midi_packet_read/write`、`tud_midi_stream_write`、`MIDI_CIN_*`。
  - `recipes/usbtmc_device.md` — USBTMC 测试测量仪器（SCPI/VISA、USB488、`*IDN?`、status byte STB/MAV/SRQ、Bulk-IN/OUT、清除/中止）。来源示例：`examples/device/usbtmc/`（含 `visaQuery.py`）。API：`tud_usbtmc_transmit_dev_msg_data`、`tud_usbtmc_get_stb_cb`。
  - `recipes/typec_power_delivery.md` — USB Type-C / PD 3.0 sink（PDO 解析、RDO 请求、Accept/PS_READY），标注 WIP 仅 STM32 G4 限制。来源示例：`examples/typec/power_delivery/`。API：`tuc_init/task/msg_request`、`pd_pdo_fixed_t`、`pd_rdo_fixed_variable_t`。
- `SKILL.md`：Scenario Quick Reference 新增上述 5 recipe（设备栈 4 个 + 新增 "Type-C / Power Delivery" 分组）；示例复制策略补充各新类的最佳起点示例；metadata.version `1.0.0` → `1.1.0`。
- `resources/api_reference.md`：新增 Audio / DFU / MIDI / USBTMC / Type-C 五节真实 API 签名（全部从头文件验证）。
- `resources/example_list.md`：补充 audio/dfu/midi/usbtmc/typec 示例的关键区分说明与 `cdc_uac2` 组合设备条目。

### Grounding
- 新配方全部基于真实源码：`src/class/{audio,dfu,midi,usbtmc}/`、`src/typec/`、`src/device/usbd.h`（描述符/尺寸宏）、各 `examples/{device,typec}/` 的 `main.c` + `tusb_config.h` + `usb_descriptors.c`。DFU 模式与 Type-C/PD 栈在 README 标注为 WIP，配方已显式声明该限制。

## [1.0.0] - 2026-06-18

### Added
- 初始发布。
- `SKILL.md`：核心原则（12 条）、When to Use、9 个配方索引、支持芯片/设备类速查、USB 设备状态机、12 条 Critical Pitfalls（含 WRONG/CORRECT 代码块）、执行工作流与失败策略。
- `AGENTS.md`：项目上下文、文件命名/include 约定、标准设备项目结构、主循环/中断/回调模板、CMake/Make 构建工作流、ESP-IDF 集成说明、代码生成 checklist、Do Not Modify 清单。
- `recipes/`：9 个场景配方
  - `device_getting_started.md`（设备栈从零集成）
  - `cdc_virtual_serial.md`（CDC 虚拟串口）
  - `hid_device.md`（HID 键盘/鼠标/手柄）
  - `msc_device.md`（MSC U 盘 / RAM 后端 / 多 LUN）
  - `descriptors_config.md`（设备/配置/字符串描述符）
  - `vendor_and_webusb.md`（Vendor 类 + WebUSB）
  - `host_cdc_msc_hid.md`（USB 主机 CDC/MSC/HID）
  - `build_and_flash.md`（CMake/Make 构建与烧录）
  - `tusb_config_guide.md`（tusb_config.h 全量配置）
- `resources/`：4 个速查文档
  - `api_reference.md`（按模块分组的真实 API/回调签名）
  - `config_reference.md`（CFG_*/OPT_* 配置宏与枚举）
  - `pitfalls.md`（按现象分类的 10 类陷阱）
  - `example_list.md`（设备/主机/双角色/Type-C 真实示例索引）
- `README.md`：中文介绍、安装方式、目录结构、支持范围、数据来源。
- `CHANGELOG.md`：本文件。

### Grounding
- 所有 API/宏/配置项/示例路径均来自 TinyUSB 真实源码（`src/`、`docs/`、`examples/`），版本对应仓库 HEAD（`docs/reference/*`、`src/tusb_option.h`、`src/class/*`、`examples/{device,host,dual,typec}/`）。
