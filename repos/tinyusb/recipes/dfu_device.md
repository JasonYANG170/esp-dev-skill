# DFU 固件升级设备类（DFU 模式 + DFU Runtime）

> **适用摘要**: 用 TinyUSB DFU 类实现 USB 固件升级，区分 **DFU Runtime**（应用运行时通过 DETACH 跳转到 bootloader）与 **DFU 模式**（ bootloader 内实际收发固件），含 download/upload/manifest 回调、多分区（alt）、bwPollTimeout 与 `tud_dfu_finish_flashing` 异步完成。

## 触发意图

- "USB 固件升级 / firmware update over USB"
- "TinyUSB DFU / DFU mode / DFU Runtime"
- "tud_dfu_download_cb / tud_dfu_upload_cb"
- "dfu-util"
- "USB DFU detach / reboot to bootloader"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/device/dfu/`（DFU 模式）、`examples/device/dfu_runtime/`（DFU Runtime） |
| 配置项 | DFU 模式：`CFG_TUD_DFU=1` + `CFG_TUD_DFU_XFER_BUFSIZE`；Runtime：`CFG_TUD_DFU_RUNTIME=1` |
| 头文件 | `src/class/dfu/dfu_device.h`（模式 API）、`src/class/dfu/dfu_rt_device.h`（Runtime API）、`src/class/dfu/dfu.h`（状态/请求/属性枚举）、`src/device/usbd.h`（`TUD_DFU_DESCRIPTOR` / `TUD_DFU_RT_DESCRIPTOR`） |
| 主机工具 | `dfu-util`（`-D` 下载、`-U` 上传、`-e` 触发 detach） |

## 分步说明

### 1. 先分清两种模式（最易混淆点）

| 维度 | DFU Runtime（`CFG_TUD_DFU_RUNTIME`） | DFU 模式（`CFG_TUD_DFU`） |
|---|---|---|
| 用途 | 应用运行时声明"我能进 DFU"，主机发 DETACH 后跳到 bootloader | bootloader 内真正接收/烧写固件 |
| 接口协议 | `DFU_PROTOCOL_RT`（0x01） | `DFU_PROTOCOL_DFU`（0x02） |
| 端点 | 无数据端点，只用 EP0 | 无数据端点，只用 EP0 |
| 描述符宏 | `TUD_DFU_RT_DESCRIPTOR()`（18 字节） | `TUD_DFU_DESCRIPTOR()`（含 N 个 alt 接口） |
| 应用回调 | `tud_dfu_runtime_reboot_to_dfu_cb()` | `tud_dfu_download_cb/upload_cb/manifest_cb/...` |
| README 状态 | 完全支持 | DFU 模式标记为 WIP，但示例可编译 |

实际产品里两者配合：正常应用开 DFU Runtime，主机 `dfu-util -e` 触发 detach → 应用 reboot → bootloader 以 DFU 模式枚举 → 主机下载新固件。`examples/device/dfu_runtime` 是最小演示（收到 detach 只改 LED 闪烁，不真跳转）；`examples/device/dfu` 是模式侧完整演示。

### 2. DFU 模式：tusb_config.h

```c
#define CFG_TUD_DFU               1
// 传输缓冲，必须与描述符里的 _xfer_size 一致
#define CFG_TUD_DFU_XFER_BUFSIZE  (TUD_OPT_HIGH_SPEED ? 512 : 64)
```

### 3. DFU 模式：必需回调（download / upload / manifest）

DFU 模式靠 EP0 类特定请求驱动，应用实现以下回调（取自 `examples/device/dfu/src/main.c`）。`alt` 即分区号（Flash/EEPROM 等多分区用不同 alt）：

```c
// 主机发 DFU_DNLOAD（wLength>0）后栈进入 DNBUSY 状态前调用
// 可同步或异步；完成后必须调 tud_dfu_finish_flashing()
void tud_dfu_download_cb(uint8_t alt, uint16_t block_num, uint8_t const *data, uint16_t length) {
  (void)alt; (void)block_num;
  // 例：打印收到的字节（真实场景里写 Flash）
  for (uint16_t i = 0; i < length; i++) printf("%c", data[i]);
  tud_dfu_finish_flashing(DFU_STATUS_OK);     // 成功；失败用 errWRITE/errVERIFY 等
}

// 主机发 DFU_DNLOAD（wLength=0）表示下载结束，栈进入 MANIFEST
// 应用可做校验或整体烧写
void tud_dfu_manifest_cb(uint8_t alt) {
  (void)alt;
  printf("Download completed, enter manifestation\r\n");
  tud_dfu_finish_flashing(DFU_STATUS_OK);
}

// 主机发 DFU_UPLOAD：把数据填进 data，返回字节数
uint16_t tud_dfu_upload_cb(uint8_t alt, uint16_t block_num, uint8_t *data, uint16_t length) {
  (void)length;
  if (block_num != 0u) return 0;   // 本示例只支持单块上传
  const uint16_t xfer_len = tu_min16((uint16_t)strlen(upload_image[alt]), length);
  memcpy(data, upload_image[alt], xfer_len);
  return xfer_len;
}
```

### 4. DFU 模式：bwPollTimeout 与异步烧写

慢速 Flash / EEPROM 不能在一个 USB 帧内烧完，需要 `bwPollTimeout` 让主机等待。返回毫秒数，期间主机不发请求：

```c
uint32_t tud_dfu_get_timeout_cb(uint8_t alt, uint8_t state) {
  if (state == DFU_DNBUSY) {
    return (alt == 0) ? 1 : 100;   // alt0 Flash 1ms；alt1 EEPROM 100ms
  }
  if (state == DFU_MANIFEST) return 0;   // 不缓冲整镜像则 0
  return 0;
}
```

> 异步烧写：`tud_dfu_download_cb` 可在烧写未完成时就返回，稍后在烧写完成处调 `tud_dfu_finish_flashing(DFU_STATUS_OK)`。在此之前主机收到 GETSTATUS 都会按 `bwPollTimeout` 等待。

### 5. DFU 模式：中止与 detach 回调

```c
// 主机终止上传/下载
void tud_dfu_abort_cb(uint8_t alt) {
  (void)alt;
  printf("Host aborted transfer\r\n");
}

// 主机发 DFU_DETACH（模式侧较少用，多在 Runtime 侧）
void tud_dfu_detach_cb(void) {
  printf("Host detach, we should probably reboot\r\n");
}
```

### 6. DFU 模式：描述符（多 alt 分区）

`TUD_DFU_DESCRIPTOR(itfnum, alt_count, stridx, attr, timeout, xfer_size)`。alt 用作分区（Flash/EEPROM），每个 alt 是独立的接口 alternate setting（取自 `examples/device/dfu/src/usb_descriptors.c`）：

```c
#define FUNC_ATTRS  (DFU_ATTR_CAN_UPLOAD | DFU_ATTR_CAN_DOWNLOAD | DFU_ATTR_MANIFESTATION_TOLERANT)

// 2 个 alt（2 个分区），字符串索引从 4 开始递增
TUD_DFU_DESCRIPTOR(ITF_NUM_DFU_MODE, 2, 4, FUNC_ATTRS, 1000, CFG_TUD_DFU_XFER_BUFSIZE),
```

属性位（`dfu.h`）：`DFU_ATTR_CAN_DOWNLOAD`、`DFU_ATTR_CAN_UPLOAD`、`DFU_ATTR_MANIFESTATION_TOLERANT`、`DFU_ATTR_WILL_DETACH`。

### 7. DFU Runtime：配置与唯一回调

```c
// tusb_config.h
#define CFG_TUD_DFU_RUNTIME  1
```

Runtime 侧只需实现一个回调——主机发 DETACH 时栈调用它（前提是描述符里 `DFU_ATTR_WILL_DETACH` 置位；否则栈等 USB 复位）：

```c
// 收到 DFU_DETACH：真实产品里在此 reboot 进 bootloader
void tud_dfu_runtime_reboot_to_dfu_cb(void) {
  // examples/device/dfu_runtime 仅改 LED 闪烁作演示：
  blink_interval_ms = BLINK_DFU_MODE;
  // 实际：NVIC_SystemReset(); 或触发看门狗复位进 bootloader
}
```

### 8. DFU Runtime：描述符

```c
// TUD_DFU_RT_DESC_LEN = 18（9 接口 + 9 功能）
TUD_DFU_RT_DESCRIPTOR(ITF_NUM_DFU_RT, 0, FUNC_ATTRS, 1000, CFG_TUD_DFU_XFER_BUFSIZE),
```

### 9. DFU 状态机（`dfu.h` 枚举）

```
APP_IDLE ⇄ APP_DETACH      （Runtime 侧）
DFU_IDLE → DFU_DNLOAD_SYNC → DFU_DNBUSY → DFU_DNLOAD_IDLE ⇄ ...（下载）
        → DFU_MANIFEST_SYNC → DFU_MANIFEST → DFU_MANIFEST_WAIT_RESET
DFU_IDLE → DFU_UPLOAD_IDLE ⇄ ...（上传）
任一状态出错 → DFU_ERROR（需 CLRSTATUS 复位）
```

`tud_dfu_get_timeout_cb` 的 `state` 参数即取自上述 `dfu_state_t`（如 `DFU_DNBUSY`、`DFU_MANIFEST`）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `dfu-util -l` 列不出设备 | DFU 接口描述符 subclass/protocol 错 | 模式用 `DFU_PROTOCOL_DFU`、Runtime 用 `DFU_PROTOCOL_RT`；subclass 都是 `APP_SUBCLASS_DFU_RUNTIME`（`TUD_DFU_APP_SUBCLASS`） |
| `dfu-util` 报 `Cannot open DFU device` | udev 权限 / 驱动 | Linux 加 udev 规则；Windows 用 libusb 驱动（Zadig） |
| 下载卡住不动 | `tud_dfu_finish_flashing` 没调 | download/manifest 回调里完成烧写后必须调用，否则栈永远停在 DNBUSY/MANIFEST |
| 慢 Flash 超时 | `tud_dfu_get_timeout_cb` 返回 0 | 按真实烧写耗时返回 `bwPollTimeout`（ms） |
| Runtime 收到 detach 不跳转 | 只实现空回调 | 在 `tud_dfu_runtime_reboot_to_dfu_cb` 里真实复位进 bootloader |
| 多分区下载到错分区 | 没按 alt 区分后端 | 回调里的 `alt` 参数即分区号，按 `alt` 路由到 Flash/EEPROM |
| `#error CFG_TUD_DFU_XFER_BUFSIZE must be defined` | 模式侧必填 | 必须等于描述符 `_xfer_size`（FS 通常 64，HS 512） |
| detach 后主机仍连着 | 描述符未设 `DFU_ATTR_WILL_DETACH` | 置位后栈主动 detach；否则等主机复位总线 |

## 参考

- `examples/device/dfu/src/main.c` — DFU 模式：download/upload/manifest/abort 完整回调
- `examples/device/dfu/src/usb_descriptors.c` — 多 alt（分区）DFU 模式描述符
- `examples/device/dfu_runtime/src/main.c` — DFU Runtime：detach 回调演示
- `src/class/dfu/dfu_device.h` — DFU 模式 API（`tud_dfu_finish_flashing`）与回调
- `src/class/dfu/dfu_rt_device.h` — DFU Runtime API（`tud_dfu_runtime_reboot_to_dfu_cb`）
- `src/class/dfu/dfu.h` — 状态机（`dfu_state_t`）、请求、状态码（`DFU_STATUS_*`）、属性位（`DFU_ATTR_*`）
- `src/device/usbd.h` — `TUD_DFU_DESCRIPTOR` / `TUD_DFU_RT_DESCRIPTOR` / `TUD_DFU_ALT_*`
