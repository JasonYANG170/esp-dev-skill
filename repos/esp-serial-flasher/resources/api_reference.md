# ESP Serial Flasher 公共 API 速查

> 全部签名来自 `include/esp_loader.h`、`include/esp_loader_io.h`、`include/esp_loader_error.h`（v2 API）。私有头（`private_include/`）不在此列，勿直接使用。

## 错误码（`esp_loader_error.h`）

```c
typedef enum {
    ESP_LOADER_SUCCESS,                // 成功
    ESP_LOADER_ERROR_FAIL,             // 未指定错误
    ESP_LOADER_ERROR_TIMEOUT,          // 超时
    ESP_LOADER_ERROR_IMAGE_SIZE,       // 镜像大于 flash 容量 / 越界
    ESP_LOADER_ERROR_INVALID_MD5,      // MD5 不匹配
    ESP_LOADER_ERROR_INVALID_PARAM,    // 参数非法（未对齐等）
    ESP_LOADER_ERROR_INVALID_TARGET,   // 目标非法 / 不支持
    ESP_LOADER_ERROR_UNSUPPORTED_CHIP, // 芯片不在支持矩阵
    ESP_LOADER_ERROR_UNSUPPORTED_FUNC, // 该协议/目标不支持此功能
    ESP_LOADER_ERROR_INVALID_RESPONSE  // 内部错误
} esp_loader_error_t;
```

便捷宏（`esp_loader.h`）：
```c
#define RETURN_ON_ERROR(x) do { esp_loader_error_t _err_ = (x); \
    if (_err_ != ESP_LOADER_SUCCESS) return _err_; } while(0)
```

## 目标芯片枚举（`esp_loader.h`）

```c
typedef enum {
    ESP8266_CHIP = 0, ESP32_CHIP, ESP32S2_CHIP, ESP32C3_CHIP, ESP32S3_CHIP,
    ESP32C2_CHIP, ESP32C5_CHIP, ESP32H2_CHIP, ESP32C6_CHIP, ESP32P4_CHIP,
    ESP32C61_CHIP, ESP_MAX_CHIP, ESP_UNKNOWN_CHIP
} target_chip_t;
```

## 协议类型（内部）

```c
typedef enum {
    ESP_LOADER_PROTOCOL_SERIAL, // SLIP 字节流：UART / USB CDC-ACM / Linux tty
    ESP_LOADER_PROTOCOL_SPI,
    ESP_LOADER_PROTOCOL_SDIO,
} esp_loader_protocol_t;
```

## Loader 上下文与初始化

```c
typedef struct esp_loader {
    /* 所有字段以 _ 前缀，私有，勿直接访问 */
    const struct esp_loader_protocol_ops_s *_protocol;
    esp_loader_port_t                      *_port;
    esp_loader_protocol_t                   _protocol_type;
    target_chip_t                           _target;
    const struct target_registers_t        *_reg;
    uint32_t  _target_flash_size;
    bool      _stub_running;
    bool      _spi_attached;
    union { struct { uint32_t sip_seq_tx; uint32_t mem_offset; } sdio;
            struct { uint8_t slave_seq_tx; uint8_t slave_seq_rx; } spi; } _proto_ctx;
} esp_loader_t;

// 选择协议并绑定 port（自动调用 port->ops->init）
esp_loader_error_t esp_loader_init_serial(esp_loader_t *loader, esp_loader_port_t *port); // UART/USB/tty
esp_loader_error_t esp_loader_init_spi   (esp_loader_t *loader, esp_loader_port_t *port); // 仅 RAM 下载
esp_loader_error_t esp_loader_init_sdio  (esp_loader_t *loader, esp_loader_port_t *port); // 实验，自动 stub

// 释放（调用 port->ops->deinit 若非 NULL，之后 loader 不可再用直到重新 init）
void esp_loader_deinit(esp_loader_t *loader);
```

## 连接

```c
typedef struct {
    uint32_t sync_timeout;  // 等待响应最大时间
    int32_t  trials;        // 连接尝试次数；>1 时每次间隔 100ms
} esp_loader_connect_args_t;

#define ESP_LOADER_CONNECT_DEFAULT() { .sync_timeout = 100, .trials = 10, }

esp_loader_error_t esp_loader_connect(esp_loader_t *loader, esp_loader_connect_args_t *connect_args);
// stub 连接（仅 serial；解锁高波特率/deflate/fast read/>2MB）
esp_loader_error_t esp_loader_connect_with_stub(esp_loader_t *loader, esp_loader_connect_args_t *connect_args);
// secure download mode（仅 serial；ESP32/ESP8266 不支持；flash_size 单位字节）
esp_loader_error_t esp_loader_connect_secure_download_mode(esp_loader_t *loader,
        esp_loader_connect_args_t *connect_args, uint32_t flash_size);

target_chip_t esp_loader_get_target(esp_loader_t *loader);  // 连接成功后才可调用
```

## 改速率

```c
// 仅 serial；连接后；ESP8266 与 SDIO 不支持
esp_loader_error_t esp_loader_change_transmission_rate(esp_loader_t *loader, uint32_t transmission_rate);
```

## Flash 烧录（明文）

```c
typedef struct {
    uint32_t offset;      // flash 地址，4 字节对齐
    uint32_t image_size;  // 镜像总大小，4 字节对齐
    uint32_t block_size;  // 每次 write 的块大小
    bool     skip_verify; // true 则 finish 跳过 MD5 校验（默认 false）
    struct { uint32_t _sequence_number; struct MD5Context _md5_context; } _state; // 库管理，勿改
} esp_loader_flash_cfg_t;

esp_loader_error_t esp_loader_flash_start (esp_loader_t *loader, esp_loader_flash_cfg_t *cfg); // 擦区间+初始化MD5
esp_loader_error_t esp_loader_flash_write (esp_loader_t *loader, esp_loader_flash_cfg_t *cfg, const void *payload, uint32_t size);
esp_loader_error_t esp_loader_flash_finish(esp_loader_t *loader, esp_loader_flash_cfg_t *cfg); // MD5校验+flash-end，必须调用
```

## Flash 烧录（deflate 压缩，仅 serial + stub；SDIO 自动 stub）

```c
typedef struct {
    uint32_t offset;           // flash 地址，4 字节对齐
    uint32_t image_size;       // 未压缩大小
    uint32_t compressed_size;  // 压缩大小
    uint32_t block_size;       // 每个压缩块大小
    struct { uint32_t _sequence_number; } _state;
} esp_loader_flash_deflate_cfg_t;

esp_loader_error_t esp_loader_flash_deflate_start (esp_loader_t *loader, esp_loader_flash_deflate_cfg_t *cfg);
esp_loader_error_t esp_loader_flash_deflate_write (esp_loader_t *loader, esp_loader_flash_deflate_cfg_t *cfg, void *payload, uint32_t size);
esp_loader_error_t esp_loader_flash_deflate_finish(esp_loader_t *loader, esp_loader_flash_deflate_cfg_t *cfg);
// 注意：deflate 路径不内部做 MD5，需用 esp_loader_flash_verify_known_md5 校验明文
```

## Flash 读 / 擦除 / 容量 / 校验

```c
// 仅 serial
esp_loader_error_t esp_loader_flash_read(esp_loader_t *loader, uint8_t *buf, uint32_t address, uint32_t length);

esp_loader_error_t esp_loader_flash_erase       (esp_loader_t *loader);                                  // 整片
esp_loader_error_t esp_loader_flash_erase_region(esp_loader_t *loader, uint32_t offset, uint32_t size);  // 均 4KB 对齐

esp_loader_error_t esp_loader_flash_detect_size(esp_loader_t *loader, uint32_t *flash_size);             // 字节

// 已知 MD5 校验（address,size,expected_md5 16字节）
esp_loader_error_t esp_loader_flash_verify_known_md5(esp_loader_t *loader, uint32_t address,
        uint32_t size, const uint8_t *expected_md5);
```

## RAM 下载

```c
typedef struct {
    uint32_t offset;     // RAM 地址
    uint32_t size;       // 总大小
    uint32_t block_size; // 每块大小
    struct { uint32_t _sequence_number; } _state;
} esp_loader_mem_cfg_t;

// 镜像头（8 字节公共头）
typedef struct { uint8_t magic; uint8_t segments; uint8_t flash_mode; uint8_t flash_size_freq; uint32_t entrypoint; } esp_loader_bin_header_t;
typedef struct { uint32_t addr; uint32_t size; const uint8_t *data; } esp_loader_bin_segment_t;

esp_loader_error_t esp_loader_mem_start (esp_loader_t *loader, esp_loader_mem_cfg_t *cfg);
esp_loader_error_t esp_loader_mem_write (esp_loader_t *loader, esp_loader_mem_cfg_t *cfg, const void *payload, uint32_t size);
esp_loader_error_t esp_loader_mem_finish(esp_loader_t *loader, esp_loader_mem_cfg_t *cfg, uint32_t entrypoint); // 跳转执行
```

## 信息读取

```c
esp_loader_error_t esp_loader_read_mac(esp_loader_t *loader, uint8_t *mac);  // 6 字节

typedef struct {
    target_chip_t target_chip;
    uint32_t eco_version;                              // ESP32-S2 无
    bool secure_boot_enabled;
    bool secure_boot_aggressive_revoke_enabled;
    bool secure_download_mode_enabled;
    bool secure_boot_revoked_keys[3];
    bool jtag_software_disabled;
    bool jtag_hardware_disabled;
    bool usb_disabled;
    bool flash_encryption_enabled;
    bool dcache_in_uart_download_disabled;
    bool icache_in_uart_download_disabled;
} esp_loader_target_security_info_t;

esp_loader_error_t esp_loader_get_security_info(esp_loader_t *loader, esp_loader_target_security_info_t *security_info); // 仅 serial��ESP32/8266 不支持

esp_loader_error_t esp_loader_read_register (esp_loader_t *loader, uint32_t address, uint32_t *reg_value);
esp_loader_error_t esp_loader_write_register(esp_loader_t *loader, uint32_t address, uint32_t reg_value);
```

## 复位

```c
void esp_loader_reset_target(esp_loader_t *loader);  // 翻转 reset 引脚
```

## Port vtable（`esp_loader_io.h`）

```c
typedef struct esp_loader_port_s { const esp_loader_port_ops_t *ops; } esp_loader_port_t;

#ifndef container_of
#define container_of(ptr, type, member) ((type *)((char *)(ptr) - offsetof(type, member)))
#endif

typedef struct {
    esp_loader_error_t (*init)(esp_loader_port_t *port);              // NULL = 无需 init
    void               (*deinit)(esp_loader_port_t *port);            // NULL = 无需 deinit
    void               (*enter_bootloader)(esp_loader_port_t *port);
    void               (*reset_target)(esp_loader_port_t *port);
    void               (*start_timer)(esp_loader_port_t *port, uint32_t ms);
    uint32_t           (*remaining_time)(esp_loader_port_t *port);     // 0 = 已超时
    void               (*delay_ms)(esp_loader_port_t *port, uint32_t ms);
    void               (*log)(esp_loader_port_t *port, esp_loader_log_level_t level, const char *fmt, va_list args);   // NULL = 静默
    void               (*log_hex)(esp_loader_port_t *port, esp_loader_log_level_t level, const char *label, const uint8_t *data, size_t size);
    esp_loader_error_t (*change_transmission_rate)(esp_loader_port_t *port, uint32_t rate);   // SDIO 置 NULL
    esp_loader_error_t (*write)(esp_loader_port_t *port, const uint8_t *data, uint16_t size, uint32_t timeout);  // SDIO 用 sdio_*
    esp_loader_error_t (*read)(esp_loader_port_t *port, uint8_t *data, uint16_t size, uint32_t timeout);
    void               (*spi_set_cs)(esp_loader_port_t *port, uint32_t level);                 // 非 SPI NULL
    esp_loader_error_t (*sdio_write)(esp_loader_port_t *port, uint32_t function, uint32_t addr, const uint8_t *data, uint16_t size, uint32_t timeout);
    esp_loader_error_t (*sdio_read)(esp_loader_port_t *port, uint32_t function, uint32_t addr, uint8_t *data, uint16_t size, uint32_t timeout);
    esp_loader_error_t (*sdio_card_init)(esp_loader_port_t *port);
} esp_loader_port_ops_t;
```

日志级别（`esp_loader_io.h`）：
```c
#define ESP_LOADER_LOG_NONE 0 / ERROR 1 / WARN 2 / INFO 3 / DEBUG 4
typedef enum { ESP_LOADER_LOG_LEVEL_NONE=0, _ERROR=1, _WARN=2, _INFO=3, _DEBUG=4 } esp_loader_log_level_t;
```

## 各平台 port 实例与 ops 符号

| 平台 / 接口 | port 结构体 | ops 符号 | 头文件 |
|---|---|---|---|
| ESP32 UART | `esp32_port_t` | `esp32_uart_ops` | `port/esp32_port.h` |
| ESP32 SPI | `esp32_spi_port_t` | `esp32_spi_ops` | `port/esp32_spi_port.h` |
| ESP32 SDIO | `esp32_sdio_port_t` | `esp32_sdio_ops` | `port/esp32_sdio_port.h` |
| ESP32 USB CDC-ACM | `esp32_usb_cdc_acm_port_t` | `esp32_usb_cdc_acm_ops` | `port/esp32_usb_cdc_acm_port.h` |
| Linux | `linux_port_t` | `linux_uart_ops` | `port/linux_port.h` |
| Zephyr | `zephyr_port_t` | （经 `esp_loader_from_device`，不手填） | `port/zephyr_port.h` |
| Raspberry Pi Pico | `pi_pico_port_t` | `pi_pico_uart_ops` | `port/pi_pico_port.h` |
| STM32 | `stm32_port_t` | `stm32_uart_ops` | `port/stm32_port.h` |

### ESP32 UART port 字段（`esp32_port_t`）

```c
typedef struct {
    esp_loader_port_t port;
    uint32_t      baud_rate;
    uint32_t      uart_port;          // UART_NUM_x
    gpio_num_t    uart_rx_pin, uart_tx_pin;
    gpio_num_t    reset_pin, boot_pin;
    uint32_t      rx_buffer_size;     // 0 = 默认 400
    uint32_t      tx_buffer_size;     // 0 = 默认 400
    uint32_t      queue_size;         // 0 = 无队列
    QueueHandle_t *uart_queue;        // 非 NULL 则写入句柄
    bool          dont_initialize_peripheral;
    int64_t       _time_end;
    bool          _peripheral_needs_deinit;
} esp32_port_t;
```

### ESP32 SPI port 字段（`esp32_spi_port_t`）

```c
typedef struct {
    esp_loader_port_t port;
    spi_host_device_t spi_bus;
    uint32_t          frequency;
    gpio_num_t spi_clk_pin, spi_miso_pin, spi_mosi_pin, spi_cs_pin;
    gpio_num_t spi_quadwp_pin, spi_quadhd_pin;
    gpio_num_t reset_pin;
    gpio_num_t strap_bit0_pin, strap_bit1_pin, strap_bit2_pin, strap_bit3_pin;
    bool dont_initialize_bus;
    /* 私有 */
} esp32_spi_port_t;
```

### ESP32 SDIO port 字段（`esp32_sdio_port_t`）

```c
typedef enum { SDIO_4BIT = 0, SDIO_1BIT } sdio_bus_width_t;

typedef struct {
    esp_loader_port_t port;
    int              slot;            // SDMMC_HOST_SLOT_1
    uint32_t         max_freq_khz;    // SDMMC_FREQ_DEFAULT 等
    gpio_num_t sdio_clk_pin, sdio_d0_pin, sdio_d1_pin, sdio_d2_pin, sdio_d3_pin, sdio_cmd_pin;
    gpio_num_t reset_pin, boot_pin;
    bool dont_initialize_host_driver;
    sdio_bus_width_t bus_width;
    /* 私有 */
} esp32_sdio_port_t;

esp_loader_error_t loader_port_wait_int(esp32_sdio_port_t *port, uint32_t timeout);
```

### ESP32 USB CDC-ACM port 字段（`esp32_usb_cdc_acm_port_t`）

```c
#define USB_VID_PID_AUTO_DETECT  (0)
#define ESPRESSIF_VID             (0x303a)
#define ESP_SERIAL_JTAG_PID       (0x1001)
#define SILICON_LABS_VID          (0x10C4)
#define CP210X_PID (0xEA60) / CP2105_PID (0xEA70) / CP2108_PID (0xEA71)
#define NANJING_QINHENG_MICROE_VID (0x1A86)
#define CH340_PID (0x7522) / CH340_PID_1 (0x7523) / CH341_PID (0x5523)

typedef void (*loader_port_esp32_usb_cdc_acm_callback_t)(void);

typedef struct {
    esp_loader_port_t port;
    uint16_t device_vid, device_pid;
    uint32_t connection_timeout_ms;
    uint32_t out_buffer_size;   // 须 > 最大 USB 包
    loader_port_esp32_usb_cdc_acm_callback_t acm_host_error_callback;
    loader_port_esp32_usb_cdc_acm_callback_t device_disconnected_callback;
    loader_port_esp32_usb_cdc_acm_callback_t acm_host_serial_state_callback;
    /* 私有 */
} esp32_usb_cdc_acm_port_t;
```

### Linux port 字段（`linux_port_t`）

```c
typedef enum { LINUX_GPIO_NONE, LINUX_GPIO_GPIOD, LINUX_GPIO_DTR_RTS } linux_gpio_mode_t;

typedef struct {
    esp_loader_port_t port;
    const char       *device;          // "/dev/ttyUSB0"
    uint32_t          baudrate;
    linux_gpio_mode_t gpio_mode;
    const char       *gpio_chip_path;  // 仅 GPIOD
    uint32_t          reset_pin;       // 仅 GPIOD
    uint32_t          boot_pin;        // 仅 GPIOD
    /* 私有 */
} linux_port_t;
```

### Zephyr 用法（DTS 驱动，非手填）

```c
// 来自 device tree 的 esp-loader 节点（chosen: zephyr,esp-loader）
const struct device *dev = DEVICE_DT_GET(DT_CHOSEN(zephyr_esp_loader));

esp_loader_t                    *loader = esp_loader_from_device(dev);
const esp_loader_config_t       *config = esp_loader_config_from_device(dev);
const esp_loader_connect_args_t *args   = esp_loader_connect_args_from_device(dev);

esp_loader_connect(loader, (esp_loader_connect_args_t *)args);

// 设置/恢复主机端 UART 波特率（与目标的 change_transmission_rate 不同）
esp_loader_error_t esp_loader_host_baudrate(esp_loader_t *loader, uint32_t baudrate);
```

`esp_loader_config_t`（`port/zephyr_port.h`，由 DTS 填充）：
```c
typedef struct {
    const struct device  *uart_dev;
    struct gpio_dt_spec   enable_spec;   // reset-gpios
    struct gpio_dt_spec   boot_spec;     // boot-gpios
    uint32_t              baud_rate;      // default-baudrate
    uint32_t              baud_rate_high; // higher-baudrate（0 = 不提速）
} esp_loader_config_t;
```

DTS 绑定（`zephyr/dts/bindings/misc/esp-loader/espressif,esp-loader.yaml`）必填属性：`uart`(phandle)、`reset-gpios`、`boot-gpios`、`default-baudrate`、`higher-baudrate`、`num-trials`、`sync-timeout-ms`。

### STM32 port 字段（`stm32_port_t`）— 预初始化外设模型

```c
// port/stm32_port.h —— init 回调为 NULL，外设必须由 STM32CubeMX 预先生成并初始化
typedef struct {
    esp_loader_port_t   port;
    UART_HandleTypeDef *huart;           // CubeMX 生成的 UART 句柄
    GPIO_TypeDef       *port_boot;       // TARGET_BOOT_GPIO_Port（CubeMX user label）
    uint16_t            pin_num_boot;    // TARGET_BOOT_Pin
    GPIO_TypeDef       *port_rst;        // TARGET_RESET_GPIO_Port
    uint16_t            pin_num_rst;     // TARGET_RESET_Pin
    uint32_t _time_end;                  // 私有
} stm32_port_t;

extern const esp_loader_port_ops_t stm32_uart_ops;
```

HAL 系列由 `__has_include("stm32xxxx_hal.h")` 自动探测（C0/F0/F1/F2/F3/F4/F7/G0/G4/H5/H7/L0/L1/L4/L5/U0/U5/WB/WL 全覆盖）。

### Raspberry Pi Pico port 字段（`pi_pico_port_t`）

```c
// port/pi_pico_port.h —— init 回调自动初始化 UART/GPIO
typedef struct {
    esp_loader_port_t port;
    uart_inst_t *uart_inst;              // uart0 / uart1
    uint         baudrate;               // 115200
    uint         uart_rx_pin_num;        // GP21
    uint         uart_tx_pin_num;        // GP20
    uint         reset_pin_num;          // GP19
    uint         boot_pin_num;           // GP18
    bool         dont_initialize_peripheral;  // true = UART 已外部初始化
    /* 私有 */
} pi_pico_port_t;

extern const esp_loader_port_ops_t pi_pico_uart_ops;
```

## 示例公共助手（`examples/common/example_common.{c,h}`，非库 API 但可直接复用）

```c
#define PARTITION_TABLE_ADDRESS  0x8000
#define APPLICATION_ADDRESS      0x10000

esp_loader_error_t connect_to_target           (esp_loader_t *loader, uint32_t higher_transmission_rate);
esp_loader_error_t connect_to_target_with_stub (esp_loader_t *loader, uint32_t higher_transmission_rate);
esp_loader_error_t flash_binary                (esp_loader_t *loader, const uint8_t *bin, size_t size, size_t address);
esp_loader_error_t load_ram_binary             (esp_loader_t *loader, const uint8_t *bin);
uint32_t           get_bootloader_address      (target_chip_t chip);
```

`bootloader_addresses[]`（`example_common.c`）：ESP8266=0x0, ESP32=0x1000, S2=0x1000, C3=0x0, S3=0x0, C2=0x0, C5=0x2000, H2=0x0, C6=0x0, P4=0x2000, C61=0x0。RAM 块大小 `ESP_RAM_BLOCK = 0x1800`，bin 头 `BIN_HEADER_SIZE=0x8`、`BIN_HEADER_EXT_SIZE=0x18`。
