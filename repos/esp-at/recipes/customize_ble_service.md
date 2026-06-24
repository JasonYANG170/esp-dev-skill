# 自定义 BLE GATT 服务

> **适用摘要**: 修改 ESP-AT 的 BLE GATT 服务定义文件 `gatts_data.csv`，自定义服务/特征/描述符的 UUID、权限与值，无需改源码。

## 触发意图

- "自定义 BLE 服务"
- "修��� GATT 服务"
- "gatts_data.csv"
- "BLE 特征"
- "AT BLE 服务端"

## 前置条件

| 条件 | 要求 |
|---|---|
| 芯片 | 支持 BLE 的型号（ESP32、ESP32-C3、ESP32-C5、ESP32-C6、ESP32-C61；ESP32-C2 默认 `AT_BLE_COMMAND_SUPPORT=n`） |
| Kconfig | `CONFIG_AT_BLE_COMMAND_SUPPORT=y`（依赖 `BT_ENABLED`） |
| 分区 | `at_customize.bin` 已烧录（BLE 服务端功能依赖它） |
| 服务源文件 | `components/customized_partitions/raw_data/ble_data/gatts_data.csv` |

## 分步说明

ESP-AT 的 BLE 服务以 GATT 结构多元数组定义，源文件位于 `components/customized_partitions/raw_data/ble_data/gatts_data.csv`。每行字段：

| 字段 | 含义 |
|---|---|
| index | 序号 |
| uuid_len | UUID 长度（通常 16） |
| uuid | UUID（如 `0x2800` 主服务、`0x2803` 特征声明、`0x2901` 描述符） |
| perm | 权限/属性字节（语义随行类型不同） |
| val_max_len | 值最大长度 |
| val_cur_len | 值当前长度 |
| value | 值（hex） |

### 默认服务结构（来自 gatts_data.csv）

| index | uuid | 含义 |
|---|---|---|
| 0 | 0x2800 | 主服务定义，value=`A002`（服务 UUID） |
| 1 | 0x2803 | 特征声明，perm 为特征属性字节 |
| 2 | 0xC300 | 特征值（特征 UUID） |
| 3 | 0x2901 | 特征的用户描述描述符 |

### perm 字段两种语义（关键）

**第二行（UUID=0x2803 特征声明）的 perm 是特征属性位**，不是访问权限：

| Bit | 属性 |
|---|---|
| 0 | BROADCAST |
| 1 | READ |
| 2 | WRITE WITHOUT RESPONSE |
| 3 | WRITE |
| 4 | NOTIFY |
| 5 | INDICATE |
| 6 | AUTHENTICATION SIGNED WRITES |
| 7 | EXTENDED PROPERTIES |

默认值 `2`（bit1=READ）。例如要支持 Read+Notify，perm 设 `0x12`（bit1+bit4）。

**访问权限**（ESP_GATT_PERM_*，用于特征值行）：

```c
ESP_GATT_PERM_READ             0x0001
ESP_GATT_PERM_READ_ENCRYPTED   0x0002
ESP_GATT_PERM_WRITE            0x0010
ESP_GATT_PERM_WRITE_ENCRYPTED  0x0020
ESP_GATT_PERM_WRITE_SIGNED     0x0080
// ... 见文档与 bta_gatt_api.h
```

### 修改步骤

1. 打开 `components/customized_partitions/raw_data/ble_data/gatts_data.csv`。
2. 编辑服务/特征/描述符行：改 UUID、perm、value 等。
3. 重新编译生成 `ble_data.bin`（位于 `build/customized_partitions/`）。
4. 烧录 `ble_data.bin` 到 `at_customize.bin` 对应的 ble 数据分区地址（用 `./build.py flash` 一并烧录）。

> 也可用 `at_override_module_config` 覆盖 `gatts_data.csv`，不改 esp-at 源码（见 `recipes/override_module_config.md`）。

### 验证

```
AT+BLEINIT=1            # 初始化为 BLE 服务端
AT+BLEADVSTART          # 开始广播
```

用手机（如 nRF Connect）连接并查询服务，应看到自定义的服务/特征。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| BLE 指令不可用 | `AT_BLE_COMMAND_SUPPORT` 未开 | menuconfig 开启，依赖 `BT_ENABLED` |
| 查不到服务 | `ble_data.bin`/`at_customize.bin` 未烧 | `./build.py flash` 一并烧录 |
| 特征属性不符预期 | 把 0x2803 行 perm 当成访问权限 | 该行 perm 是属性位（READ/NOTIFY 等），见上表 |
| 访问权限错误 | 用了属性位当访问权限 | 访问权限用 `ESP_GATT_PERM_*`（0x0001/0x0010...） |
| ESP32-C2 无 BLE 指令 | C2 默认 `AT_BLE_COMMAND_SUPPORT=n` | 需手动开启且依赖 NimBLE 配置 |
| UUID 长度错 | uuid_len 与 uuid 字节数不匹配 | 16 字节 UUID 用 `uuid_len=16` |

## 参考

- 仓库文档：`docs/en/Compile_and_Develop/How_to_customize_BLE_services.rst`
- 服务源文件：`components/customized_partitions/raw_data/ble_data/gatts_data.csv`
- 覆盖方式：`examples/at_override_module_config/`、`recipes/override_module_config.md`
