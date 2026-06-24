# ESP-WDF WASM 应用 API 速查手册

> 本手册中的所有函数签名、结构体、宏均来自 esp-wdf 仓库的真实头文件（`components/wamr/app-framework/include/`、`components/wamr/libc-builtin-extended/include/ioctl/`、`components/extended_wasm_app/`）。未在仓库中找到的 API 一律不列出。

## 概览：WASM 应用两种入口形态

ESP-WDF 中的 WebAssembly 应用有两类入口，二者互斥：

| 形态 | 入口符号 | 来源 Kconfig | 典型场景 |
|---|---|---|---|
| 标准 WASI 应用 | `int main(void)` / `int main(int argc, char *argv[])` | 默认 | hello_world、coremark、peripherals、sockets、multi_thread |
| WAMR App Framework 应用 | `on_init()` + `on_destroy()` (+可选 `on_request`/`on_response`/`on_timer_callback`) | `CONFIG_WAMR_APP_FRAMEWORK=y` | timer、event、request 等简单示例 |

当 `CONFIG_WAMR_APP_FRAMEWORK=y` 时，构建系统（`CMakeLists.txt`）会自动添加 `-Wl,--export=on_init` 和 `-Wl,--export=on_destroy`；进一步可启用 `CONFIG_WAMR_APP_FRAMEWORK_EXPORT_REQUEST_RESPONSE`（导出 `on_request`/`on_response`）与 `CONFIG_WAMR_APP_FRAMEWORK_EXPORT_TIMER`（导出 `on_timer_callback`）。

---

## 一、WAMR App Framework API

头文件：`wasm_app.h`、`wa-inc/request.h`、`wa-inc/timer_wasm_app.h`、`bi-inc/attr_container.h`、`bi-inc/shared_utils.h`

### 1.1 应用生命周期回调

WASM 应用需自行实现以下符号（由宿主在对应时机调用）：

```c
void on_init(void);     // 应用启动时调用，做初始化与资源注册
void on_destroy(void);  // 应用卸载时调用（实际清理多由库版本完成）
```

### 1.2 定时器 API（timer_wasm_app.h）

```c
typedef struct user_timer *user_timer_t;
typedef void (*on_user_timer_update_f)(user_timer_t timer);

// 创建定时器；interval 单位毫秒；is_period 是否周期；auto_start 是否立即启动
user_timer_t api_timer_create(int interval, bool is_period, bool auto_start,
                              on_user_timer_update_f on_timer_update);

// 取消定时器
void api_timer_cancel(user_timer_t timer);

// 重启定时器并设置新的 interval（毫秒）
void api_timer_restart(user_timer_t timer, int interval);
```

### 1.3 请求/响应 API（request.h + shared_utils.h）

```c
// 请求/事件处理回调签名
typedef void (*request_handler_f)(request_t *request);
// 响应处理回调签名（带用户数据）
typedef void (*response_handler_f)(response_t *response, void *user_data);

// 注册资源 URL 处理器，收到对该 URL 的请求时回调 handler
bool api_register_resource_handler(const char *url, request_handler_f handler);

// 异步发送请求，响应到达时回调 response_handler（user_data 透传）
void api_send_request(request_t *request, response_handler_f response_handler,
                      void *user_data);

// 发送响应（在 request_handler 内构造 response 后调用）
void api_response_send(response_t *response);

// 发布事件：url 为事件主题，fmt 为负载格式（FMT_ATTR_CONTAINER=99）
bool api_publish_event(const char *url, int fmt, void *payload, int payload_len);

// 订阅事件：收到匹配 url 的事件时回调 handler
bool api_subscribe_event(const char *url, request_handler_f handler);
```

请求/响应构造辅助函数（shared_utils.h）：

```c
// 初始化请求结构体（request 必须非空）
request_t *init_request(request_t *request, char *url, int action,
                        int fmt, void *payload, int payload_len);

// 根据 request 生成配对的 response（response 必须非空）
response_t *make_response_for_request(request_t *request, response_t *response);

// 设置响应字段（response 必须非空）
response_t *set_response(response_t *response, int status, int fmt,
                         const char *payload, int payload_len);
```

`request_t` 字段（shared_utils.h）：`uint32 mid; char *url; int action; int fmt; void *payload; int payload_len; unsigned long sender;`

`response_t` 字段：`uint32 mid; int status; int fmt; void *payload; int payload_len; unsigned long reciever;`

### 1.4 CoAP 方法与状态码（request.h）

```c
// 方法
typedef enum { COAP_GET=1, COAP_POST, COAP_PUT, COAP_DELETE, COAP_EVENT=(COAP_DELETE+2) } coap_method_t;

// 响应状态码（节选）
CREATED_2_01 = 65;     DELETED_2_02 = 66;    VALID_2_03 = 67;
CHANGED_2_04 = 68;     CONTENT_2_05 = 69;    CONTINUE_2_31 = 95;
BAD_REQUEST_4_00 = 128; NOT_FOUND_4_04 = 132; INTERNAL_SERVER_ERROR_5_00 = 160;
```

### 1.5 负载格式宏（shared_utils.h）

```c
#define FMT_ATTR_CONTAINER 99   // attr_container_t 序列化负载
#define FMT_APP_RAW_BINARY 98   // 原始二进制负载
```

---

## 二、属性容器 attr_container API（bi-inc/attr_container.h）

用于在应用与虚拟机之间、应用与应用之间传递结构化参数。

```c
typedef struct attr_container { char flags[2]; char buf[1]; } attr_container_t;

attr_container_t *attr_container_create(const char *tag);
void attr_container_destroy(const attr_container_t *attr_cont);

// —— 写入（注意首参为二级指针，容器可能被重建）——
bool attr_container_set_string(attr_container_t **p, const char *key, const char *value);
bool attr_container_set_int(attr_container_t **p, const char *key, int value);
bool attr_container_set_int32(attr_container_t **p, const char *key, int32_t value);
bool attr_container_set_uint32(attr_container_t **p, const char *key, uint32_t value);
bool attr_container_set_int64(attr_container_t **p, const char *key, int64_t value);
bool attr_container_set_uint64(attr_container_t **p, const char *key, uint64_t value);
bool attr_container_set_short(attr_container_t **p, const char *key, short value);
bool attr_container_set_int16(attr_container_t **p, const char *key, int16_t value);
bool attr_container_set_uint16(attr_container_t **p, const char *key, uint16_t value);
bool attr_container_set_byte(attr_container_t **p, const char *key, int8_t value);
bool attr_container_set_int8(attr_container_t **p, const char *key, int8_t value);
bool attr_container_set_uint8(attr_container_t **p, const char *key, uint8_t value);
bool attr_container_set_float(attr_container_t **p, const char *key, float value);
bool attr_container_set_double(attr_container_t **p, const char *key, double value);
bool attr_container_set_bool(attr_container_t **p, const char *key, bool value);
bool attr_container_set_bytearray(attr_container_t **p, const char *key,
                                  const int8_t *value, unsigned length);

// —— 读取（找不到键时数值返回 0、字符串/字节数组返回 NULL）——
char *attr_container_get_as_string(const attr_container_t *p, const char *key);
int   attr_container_get_as_int(const attr_container_t *p, const char *key);
int32_t  attr_container_get_as_int32(const attr_container_t *p, const char *key);
uint32_t attr_container_get_as_uint32(const attr_container_t *p, const char *key);
int64_t  attr_container_get_as_int64(const attr_container_t *p, const char *key);
uint64_t attr_container_get_as_uint64(const attr_container_t *p, const char *key);
short    attr_container_get_as_short(const attr_container_t *p, const char *key);
int16_t  attr_container_get_as_int16(const attr_container_t *p, const char *key);
uint16_t attr_container_get_as_uint16(const attr_container_t *p, const char *key);
int8_t   attr_container_get_as_byte(const attr_container_t *p, const char *key);
uint8_t  attr_container_get_as_uint8(const attr_container_t *p, const char *key);
float    attr_container_get_as_float(const attr_container_t *p, const char *key);
double   attr_container_get_as_double(const attr_container_t *p, const char *key);
bool     attr_container_get_as_bool(const attr_container_t *p, const char *key);
const int8_t *attr_container_get_as_bytearray(const attr_container_t *p,
                                              const char *key, unsigned *array_length);

// —— 查询与序列化 ——
const char *attr_container_get_tag(const attr_container_t *attr_cont);
uint16_t    attr_container_get_attr_num(const attr_container_t *attr_cont);
bool        attr_container_contain_key(const attr_container_t *attr_cont, const char *key);
unsigned    attr_container_get_serialize_length(const attr_container_t *attr_cont);
bool        attr_container_serialize(char *buf, const attr_container_t *attr_cont);
bool        attr_container_is_constant(const attr_container_t *attr_cont);
void        attr_container_dump(const attr_container_t *attr_cont);  // 打印到 printf
```

属性类型枚举（attr_container.h）：`ATTR_TYPE_BYTE(0)`、`ATTR_TYPE_SHORT(1)`、`ATTR_TYPE_INT(2)`、`ATTR_TYPE_INT64(3)`、`ATTR_TYPE_UINT8(4)`、`ATTR_TYPE_UINT16(5)`、`ATTR_TYPE_UINT32(6)`、`ATTR_TYPE_UINT64(7)`、`ATTR_TYPE_FLOAT(10)`、`ATTR_TYPE_DOUBLE(11)`、`ATTR_TYPE_BOOLEAN(12)`、`ATTR_TYPE_STRING(13)`、`ATTR_TYPE_BYTEARRAY(14)`。

---

## 三、外设 VFS/ioctl API（libc-builtin-extended/include/ioctl/）

ESP-WDF 将外设映射为类 POSIX 设备节点，WASM 应用通过 `open`/`read`/`write`/`ioctl`/`close` 访问。需包含 `<fcntl.h>`、`<unistd.h>`、`<errno.h>`、`<sys/ioctl.h>` 及对应 `ioctl/esp_xxx_ioctl.h`。

### 3.1 设备节点路径

| 外设 | 设备路径 | 配置项示例 |
|---|---|---|
| GPIO | `/dev/gpio/<pin>` | `CONFIG_GPIO_SIMPLE_GPIO_PIN_NUM` |
| UART | `/dev/uart/0`、`/dev/usbserjtag` | `CONFIG_UART_DEVICE_UART0` / `CONFIG_UART_DEVICE_USB_SERIAL_JTAG_CONTROLLER` |
| I2C | `/dev/i2c/0` | `CONFIG_I2C_SDA_PIN_NUM`、`CONFIG_I2C_SCL_PIN_NUM` |
| SPI | `/dev/spi/2`、`/dev/spi/3` | `CONFIG_SPI_DEVICE_SPI2` / `CONFIG_SPI_DEVICE_SPI3` |
| LEDC | `/dev/ledc/0`、`/dev/ledc/1`、`/dev/ledc/2` | `CONFIG_LEDC_DEVICE_LEDC0/1/2` |
| 文件系统 | `/storage/<file>` | （VFS 文件） |

### 3.2 GPIO（esp_gpio_ioctl.h）

```c
#define GPIOCSCFG  _GPIOC(0x0001)   // 设置 GPIO 配置
// 配置标志
#define GPIOC_PULLDOWN_EN   (1 << 0)
#define GPIOC_PULLUP_EN     (1 << 1)
#define GPIOC_OPENDRAIN_EN  (1 << 2)

typedef struct gpioc_cfg {
    union { struct { uint32_t pulldown_en:1; uint32_t pullup_en:1; uint32_t opendrain_en:1; } flags_data;
            uint32_t flags; };
} gpioc_cfg_t;
```
操作模式：`open("/dev/gpio/<pin>", O_WRONLY)` → `ioctl(fd, GPIOCSCFG, &cfg)` → 用 `write(fd, &state, 1)` 写电平（1/0）→ `close(fd)`。

### 3.3 I2C（esp_i2c_ioctl.h）

```c
#define I2CIOCSCFG      _I2CC(0x0001)  // 设置 I2C 配置
#define I2CIOCRDWR      _I2CC(0x0002)  // 读/写（单条消息）
#define I2CIOCEXCHANGE  _I2CC(0x0003)  // 主机收发交换
// 配置标志
#define I2C_MASTER        (1 << 0)
#define I2C_SDA_PULLUP    (1 << 1)
#define I2C_SCL_PULLUP    (1 << 2)
#define I2C_ADDR_10BIT    (1 << 3)
// 消息标志
#define I2C_MSG_WRITE       (1 << 0)
#define I2C_MSG_CHECK_ACK   (1 << 1)
#define I2C_MSG_NO_START    (1 << 2)
#define I2C_MSG_NO_END      (1 << 3)
// 交换消息标志
#define I2C_EX_MSG_READ_FIRST  (1 << 0)
#define I2C_EX_MSG_CHECK_ACK   (1 << 1)
#define I2C_EX_MSG_DELAY_EN    (1 << 2)

typedef struct i2c_cfg {
    uint16_t sda_pin, scl_pin;
    union { struct { uint32_t master:1; uint32_t sda_pullup:1; uint32_t scl_pullup:1; uint32_t addr_10bit:1; } flags_data;
            uint32_t flags; };
    union { struct { uint32_t clock; } master;
            struct { uint32_t max_clock; uint16_t addr; } slave; };
} i2c_cfg_t;

typedef struct i2c_msg { /* flags, addr, buffer, size */ } i2c_msg_t;
typedef struct i2c_ex_msg { /* flags, addr, delay_ms, tx_buffer, tx_size, rx_buffer, rx_size */ } i2c_ex_msg_t;
```

**读取 GPIO 电平（输入方向）**：GPIO 节点既支持 `write(fd,&state,1)` 写输出，也支持 `read(fd,&level,1)` 读输入电平（`0`/`1`）。TT21100 示例据此轮询 READY 引脚：`open("/dev/gpio/<ready_pin>", O_RDONLY)` → `ioctl(fd, GPIOCSCFG, &cfg)`（`cfg.flags=GPIOC_PULLUP_EN`）→ `read(fd, &level, 1)`，为 `0` 即“有数据”。

```c
uint8_t level;
int n = read(gpio_fd, &level, 1);   /* n==1 表示读到 1 字节电平 */
```

**两步变长 I2C 读取模式（如 TT21100 触摸屏）**：芯片每次中断送出变长报告，首 2 字节为本份报告总长度。先读 2 字节长度字，再按该长度读报告体，两次都用 `I2CIOCRDWR`、`flags=0`（读）、同一从机地址：

```c
i2c_msg_t msg;
uint16_t length;
uint8_t buffer[256];

msg.flags = 0; msg.addr = addr; msg.buffer = (uint8_t *)&length; msg.size = sizeof(length);
ioctl(i2c_fd, I2CIOCRDWR, &msg);          /* 第一步：读长度 */

msg.flags = 0; msg.addr = addr; msg.buffer = buffer; msg.size = length;
ioctl(i2c_fd, I2CIOCRDWR, &msg);          /* 第二步：读 length 字节报告体 */
/* buffer 可直接强转为芯片协议的 __attribute__((packed)) 结构体 */
```

> 注：`GPIOC_PULLDOWN_EN`/`GPIOC_PULLUP_EN` 在 `esp_gpio_ioctl.h` 中的代码注释与位值方向相反（注释把 `1<<0` 写成“pull-up”、`1<<1` 写成“pull-down”），但位值本身正确——`GPIOC_PULLUP_EN`=`1<<1` 确为上拉。写代码按宏名用即可，不要照抄注释。

### 3.4 SPI（esp_spi_ioctl.h）

```c
#define SPIIOCSCFG      _SPIC(0x0001)  // 设置 SPI 配置
#define SPIIOCEXCHANGE  _SPIC(0x0002)  // 主机收发交换
// 配置标志
#define SPI_MASTER    (1 << 0)
#define SPI_MODE(x)   (((x) & 0x3) << 1)
#define SPI_MODE_0    SPI_MODE(0)
#define SPI_MODE_1    SPI_MODE(1)
#define SPI_MODE_2    SPI_MODE(2)
#define SPI_MODE_3    SPI_MODE(3)
#define SPI_RX_LSB    (1 << 3)
#define SPI_TX_LSB    (1 << 4)

typedef struct spi_cfg {
    uint8_t cs_pin, sclk_pin, mosi_pin, miso_pin;
    union { struct { uint32_t master:1; uint32_t mode:2; uint32_t rx_lsb:1; uint32_t tx_lsb:1; } flags_data;
            uint32_t flags; };
    union { struct { uint32_t clock; } master; };   // 时钟频率 Hz
} spi_cfg_t;

typedef struct spi_ex_msg { const void *tx_buffer; void *rx_buffer; uint32_t size; } spi_ex_msg_t;
```

### 3.5 LEDC（esp_ledc_ioctl.h）

```c
#define LEDCIOCSCFG       _LEDCC(1)  // 设置 LEDC 配置
#define LEDCIOCSSETFREQ   _LEDCC(2)  // 设置频率
#define LEDCIOCSSETDUTY   _LEDCC(3)  // 设置通道占空比
#define LEDCIOCSSETPHASE  _LEDCC(4)  // 设置通道相位
#define LEDCIOCSPAUSE     _LEDCC(5)  // 暂停
#define LEDCIOCSRESUME    _LEDCC(6)  // 恢复

typedef struct ledc_channel_cfg { uint8_t output_pin; uint32_t duty; uint32_t phase; } ledc_channel_cfg_t;
typedef struct ledc_cfg { uint32_t frequency; uint8_t channel_num; const ledc_channel_cfg_t *channel_cfg; } ledc_cfg_t;
typedef struct ledc_duty_cfg  { uint8_t channel; uint32_t duty; }  ledc_duty_cfg_t;
typedef struct ledc_phase_cfg { uint8_t channel; uint32_t phase; } ledc_phase_cfg_t;
```

---

## 四、Socket / WASI 扩展（lib-socket）

头文件：`wasi_socket_ext.h`（应用在 `__wasi__` 环境下需包含）。提供标准 BSD socket 接口：`socket`、`connect`、`bind`、`listen`、`accept`、`send`、`recv`、`setsockopt`、`getsockopt`、`getsockname`、`getpeername`、`shutdown`、`close`，以及 `inet_pton`、`htons` 等（来自 `<sys/socket.h>`、`<arpa/inet.h>`、`<netinet/in.h>`）。

---

## 五、扩展组件 WASM API（extended_wasm_app，需对应 Kconfig 开启）

### 5.1 LVGL（esp_lvgl.h，`CONFIG_WDF_EXT_WASM_APP_LVGL=y`）

由于 WASM 沙箱禁止直接解引用虚拟机分配的结构指针，ESP-WDF 提供访问器 API：

```c
bool lvgl_is_inited(void);
int  lvgl_init(void);       // 初始化 LVGL（异步，无需应用层驱动 lv_task_handler）
int  lvgl_deinit(void);
void lvgl_lock(void);       // 暂停 LVGL 调度（操作 UI 前加锁）
void lvgl_unlock(void);     // 恢复 LVGL 调度

// 读取对象内部数据（避免直接 obj->member）
int  lv_obj_get_data(const lv_obj_t *obj, int type, void *pdata, int n);
// type 取值：LV_OBJ_COORDS(0) —— 读取 obj 的坐标到 lv_area_t
void *lv_timer_get_user_data(lv_timer_t *timer);

// 绘制描述符读写访问器
int lv_draw_rect_dsc_get_data(lv_draw_rect_dsc_t *dsc, int type, void *pdata, int n);
int lv_draw_rect_dsc_set_data(lv_draw_rect_dsc_t *dsc, int type, const void *pdata, int n);
int lv_draw_dsc_base_get_data(lv_draw_dsc_base_t *dsc, int type, void *pdata, int n);
int lv_draw_dsc_base_set_data(lv_draw_dsc_base_t *dsc, int type, const void *pdata, int n);
int lv_draw_line_dsc_get_data(lv_draw_line_dsc_t *dsc, int type, void *pdata, int n);
int lv_draw_fill_dsc_get_data(const lv_draw_fill_dsc_t *dsc, int type, void *pdata, int n);
int lv_draw_label_dsc_set_data(lv_draw_label_dsc_t *dsc, int type, const void *pdata, int n);
int lv_draw_border_dsc_set_data(lv_draw_border_dsc_t *dsc, int type, const void *pdata, int n);
int lv_draw_task_get_data(lv_draw_task_t *disp, int type, void *pdata, int n);
int lv_font_get_data(const lv_font_t *font, int type, void *pdata, int n);
int lv_disp_get_data(lv_display_t *disp, void *pdata, int n);
void lv_release_variable(void);
```

LVGL 数据类型常量（esp_lvgl.h）：`LV_OBJ_COORDS=0`、`LV_OBJ_DRAW_PART_DSC_TYPE=0`、`..._PART=1`、`..._ID=2`、`..._TEXT=3`、`..._VALUE=4`、`..._P1=5`、`..._P2=6`、`..._CLIP_AREA=7`、`..._DRAW_AREA=8`、`..._RECT_DSC=9`、`..._LINE_DSC=10` 等。

### 5.2 HTTP Client（http_client_wasm_api.h，`CONFIG_WDF_EXT_WASM_APP_HTTP_CLIENT=y`）

应用层通过 ESP-IDF 风格的 `esp_http_client.h` API（`esp_http_client_init/perform/cleanup` 等）使用，底层由 WASM 适配层桥接。函数 ID 见头文件：`HTTP_CLIENT_INIT(0)`、`HTTP_CLIENT_PERFORM(0)`、`HTTP_CLIENT_CLOSE(1)`、`HTTP_CLIENT_SET_URL(0)`、`HTTP_CLIENT_SET_METHOD(1)` 等。

### 5.3 ESP-RainMaker（`CONFIG_WDF_EXT_WASM_APP_RMAKER=y`）

应用层使用 `esp_rmaker_core.h`、`esp_rmaker_standard_devices.h`、`esp_rmaker_standard_params.h` 等：`esp_rmaker_node_init`、`esp_rmaker_device_create`、`esp_rmaker_device_add_cb`、`esp_rmaker_node_add_device`、`esp_rmaker_start`、`esp_rmaker_param_update_and_report` 等（见 `examples/rainmaker/switch`）。

### 5.4 MQTT / Wi-Fi Provisioning

`CONFIG_WDF_EXT_WASM_APP_MQTT=y` 启用 `mqtt_client.h`（`esp_mqtt_client_init/subscribe/publish/...`）；`CONFIG_WDF_EXT_WASM_APP_WIFI_PROVISIONING=y` 启用配网 API（见 `examples/provisioning/wifi_prov_mgr`）。

---

## 六、通用工具（esp_common / esp_event / log）

`esp_common`、`esp_event`、`log` 组件与 ESP-IDF 对应组件兼容，应用可直接使用 `esp_log.h`（`ESP_LOGI/ESP_LOGE`、`ESP_LOG_TAG`）、`esp_event.h`（`esp_event_handler_register` 等）、`esp_err.h`（`esp_err_t`、`ESP_OK`）。版本宏见 `esp_wdf_version.h`：`ESP_WDF_VERSION_MAJOR/MINOR/PATCH`、`ESP_WDF_VERSION`、`esp_get_wdf_version()`。
