# ESP Serial Flasher 陷阱汇总

> 来自 `SKILL.md` 的 Critical Pitfalls 与各 recipe 常见错误的整合，按主题分组。每条给出错误做法与正确做法。

## 1. v2 API 范式

### 1.1 必须先 init，第一个参数永远是 loader
- ❌ v1 风格：无上下文、不 init 直接 `esp_loader_connect(&args)`
- ✅ 构造 port → `esp_loader_init_serial(&loader, &port.port)` → 每个调用传 `&loader`

### 1.2 传给 init 的是嵌入式 base，不是整个 port
- ❌ `esp_loader_init_serial(&loader, &port)`
- ✅ `esp_loader_init_serial(&loader, &port.port)`（`port` 结构体首成员必须是 `esp_loader_port_t port`）

### 1.3 不含私有头 / 旧 serial_io.h
- ❌ 包含 `private_include/*.h` 或 `include/serial_io.h`
- ✅ 只含 `esp_loader.h`（自动含 error/io 头）+ 平台 port 头

## 2. Flash 烧录

### 2.1 `esp_loader_flash_finish()` 不能省
- ❌ 只 start/write 就 reset_target —— flash-end 没发、校验被静默跳过
- ✅ start → 循环 write → **finish**（默认 MD5 校验）→ reset_target

### 2.2 `_state` 子结构不可手填
- ❌ 手填 `_state._sequence_number` / `_md5_context`
- ✅ 只填公开字段（offset/image_size/block_size/skip_verify），`_state` 由 `flash_start` 初始化

### 2.3 image_size 必须事先已知且 4 字节对齐
- ❌ 不知道总大小就 start；offset/image_size 未对齐
- ✅ 先取得镜像大小（bin2array 符号 / SD 卡 / 缓冲长度），对齐地址

### 2.4 ESP8266 特例
- ❌ 对 8266 改波特率（返回 UNSUPPORTED_FUNC）；不设 skip_verify 导致 INVALID_MD5
- ✅ 仅非 8266 才 `change_transmission_rate`；8266 设 `flash_cfg.skip_verify = true`

### 2.5 deflate 不内部做 MD5
- ❌ 以为 `flash_deflate_finish` 校验明文 MD5
- ✅ deflate 后用 `esp_loader_flash_verify_known_md5(offset, image_size, raw_md5)` 校验

### 2.6 erase_region 必须 4KB 对齐
- ❌ offset/size 非 4096 倍数（INVALID_PARAM）
- ✅ 向上取整到 4096 倍数

## 3. 连接

### 3.1 ROM vs stub 选择
- ❌ host flash 紧张却无条件 `connect_with_stub`（拉入 ~87KB）
- ✅ host 紧张用 `esp_loader_connect()`（ROM），靠 `--gc-sections` 剥 stub；需高波特率/deflate/fast read/>2MB 才用 stub

### 3.2 SDIO 自动走 stub
- ❌ 对 SDIO 调用 `connect_with_stub`（多余）或 `change_transmission_rate`（不支持）
- ✅ SDIO `esp_loader_connect()` 即自动上传 stub；不改速率

### 3.3 secure download mode
- ❌ 不传 flash_size，或对 ESP32/ESP8266 调用
- ✅ 提供真实 flash 字节数，仅支持的芯片用

### 3.4 USB CDC-ACM 前置驱动
- ❌ 直接 init_serial，未先 `usb_host_install` + `cdc_acm_host_install`
- ✅ 先装 USB Host + 事件任务 + CDC-ACM 驱动；速率参数传 0

## 4. 速率变更

### 4.1 只在 serial 接口、连接后、非 8266/SDIO
- ❌ 连接前改速率；对 SDIO 改速率
- ✅ `esp_loader_connect` 后，非 8266 才 `change_transmission_rate`

## 5. RAM 下载

### 5.1 区分 ESP8266 头长度
- ❌ 所有芯片用 0x18 偏移读 segment
- ✅ ESP8266 用 `0x8`（BIN_HEADER_SIZE），其余 `0x18`（BIN_HEADER_EXT_SIZE）

### 5.2 SPI 接口仅 RAM 下载
- ❌ `esp_loader_init_spi` 后调 flash_start/write（UNSUPPORTED_FUNC）
- ✅ SPI 只做 `mem_start/write/finish`（RAM 下载）

### 5.3 secure download 与 RAM 下载冲突
- ❌ secure download 模式下 mem_start（INVALID_PARAM）
- ✅ RAM 下载前确认未开 secure download mode

### 5.4 entrypoint 跳转
- ❌ 自行猜 entrypoint
- ✅ 用 `header->entrypoint` 喂 `esp_loader_mem_finish`

## 6. Port 移植

### 6.1 port 首成员必须是 base
- ❌ `esp_loader_port_t port` 不在结构体第一位 → `container_of` 错位
- ✅ base 放第一位

### 6.2 未实现回调必须 NULL
- ❌ 非 SDIO 也填 sdio_*；非 SPI 也填 spi_set_cs
- ✅ 按接口只填需要的，其余 NULL（SDIO 的 write/read/change_rate 置 NULL；非 SPI 的 spi_set_cs NULL；非 SDIO 的 sdio_* NULL）

### 6.3 DMA/对齐
- ❌ 直接把协议层任意缓冲喂给有对齐要求的 DMA
- ✅ port 内用对齐 bounce buffer 或平台驱动的对齐缓冲支持（如 ESP-IDF SDIO 的 `SDMMC_HOST_FLAG_ALLOC_ALIGNED_BUF`）

### 6.4 USER_DEFINED 别链内置 port
- ❌ 顶层没设 `PORT USER_DEFINED`，链入内置 port 与自写 port 冲突
- ✅ `set(PORT USER_DEFINED)`

## 7. 错误码语义

| 错误码 | 常见触发 | 处理 |
|---|---|---|
| `TIMEOUT` | 接线/供电/波特率 | 核对 TX/RX/RESET/BOOT/共地；降波特率；短缆 |
| `INVALID_RESPONSE` | 高波特率线缆/供电 | 降波特率；短缆；独立供电 |
| `INVALID_TARGET` | 不支持的芯片/版本 | 查支持矩阵 |
| `UNSUPPORTED_FUNC` | 接口不支持该功能 | SPI 不能 flash；SDIO/USB 不改速率；8266 无 stub MD5 |
| `INVALID_MD5` | 传输错误/镜像不全 | 重烧；8266 设 skip_verify |
| `INVALID_PARAM` | 未对齐 / secure flash_size 错 | 对齐地址；核对 secure 字节数 |
| `IMAGE_SIZE` | 越界 flash 末端 | 核对 address+size |
| `UNSUPPORTED_CHIP` | flash_detect_size 遇未知 chip | 用 stub 连接 |

## 8. 集成构建

### 8.1 ESP-IDF 找不到头
- ❌ 没 add-dependency 或 include 路径缺 `../../common`
- ✅ `idf.py add-dependency "espressif/esp-serial-flasher"`；`INCLUDE_DIRS` 含 example_common 目录

### 8.2 USB CDC port 灰显
- ❌ target 不支持 USB OTG
- ✅ 换 S2/S3/P4 target（`depends on SOC_USB_OTG_SUPPORTED`）

### 8.3 bin2array 未生成 md5
- ❌ 用 ESP-IDF `EMBED_FILES`（无 md5）
- ✅ 用 `examples/common/bin2array.cmake`（生成 `_bin`/`_bin_size`/`_bin_md5`）

### 8.4 Linux 权限
- ❌ 打不开 `/dev/ttyUSB*`
- ✅ `usermod -aG dialout $USER` 后重新登录

### 8.5 Linux GPIO 链接失败
- ❌ `LINUX_PORT_GPIO=ON` 但无 libgpiod ≥2.0
- ✅ `apt install libgpiod-dev`（Debian13/Ubuntu24.04+）或源码编译

## 9. 参考来源
- 仓库 `README.md`（支持矩阵、特性、stub 体积、已知限制）
- `docs/configuration.md`、`docs/hardware-connections.md`、`docs/platform-setup.md`、`docs/supporting-new-platform.md`、`docs/migration-v1-to-v2.md`
- `include/esp_loader.h`、`include/esp_loader_io.h`、`include/esp_loader_error.h`
- `examples/common/example_common.c`、各 `examples/esp32_*/main/main.c`
- `Kconfig`、`idf_component.yml`
