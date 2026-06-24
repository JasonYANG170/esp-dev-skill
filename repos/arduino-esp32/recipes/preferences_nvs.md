# Preferences (NVS 持久化存储)

> **适用摘要**: 使用 ESP32 专属 `Preferences` 库（基于片上 NVS）保存配置/计数器等小数据，含 namespace、各类型读写、只读模式。建议替代 Arduino EEPROM。

## 触发意图

- "掉电保存 / 持久化"
- "保存配置 / 计数器"
- "NVS 读写"
- "Preferences 用法"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `libraries/Preferences/examples/StartCounter/StartCounter.ino` |
| 分区表 | 默认含 `nvs` 分区（≥12 KB）；自定义分区表需自行加 `nvs, data, nvs` |
| 长度限制 | namespace ≤ 15 字符，key ≤ 15 字符 |
| 适用场景 | 大量小键值；大块数据用 LittleFS/FFat/SPIFFS |

## 分步说明

### 基本读写（启动计数器，取自仓库示例）

```cpp
#include <Preferences.h>
Preferences prefs;

void setup() {
    Serial.begin(115200);
    prefs.begin("my-app", false);                  // false=读写；true=只读
    unsigned int counter = prefs.getUInt("counter", 0);   // key 不存在返回默认值
    counter++;
    Serial.printf("boot count: %u\n", counter);
    prefs.putUInt("counter", counter);
    prefs.end();
}
void loop() {}
```

### 各数据类型

支持的类型与对应方法（节选，详见 `docs/en/api/preferences.rst`）：

| 类型 | put | get |
|---|---|---|
| bool | `putBool` | `getBool(key, false)` |
| int8/uint8 | `putChar`/`putUChar` | `getChar`/`getUChar` |
| int16/uint16 | `putShort`/`putUShort` | `getShort`/`getUShort` |
| int32/uint32 | `putInt`/`putUInt`/`putLong`/`putULong` | `getInt`/`getUInt`/... |
| int64/uint64 | `putLong64`/`putULong64` | `getLong64`/`getULong64` |
| float/double | `putFloat`/`putDouble` | `getFloat(key, NAN)`/`getDouble` |
| 字符串 | `putString(key, "x")` 或 `String` | `getString(key, buf, maxLen)` / `String getString(key, default)` |
| 任意字节 | `putBytes(key, ptr, len)` | `getBytes(key, buf, maxLen)` |

辅助：`getStringLength(key)` / `getBytesLength(key)`（含结尾 `\0`，用于预先分配缓冲）。

### 管理键

```cpp
prefs.isKey("counter");     // 是否存在
prefs.remove("counter");    // 删单个 key
prefs.clear();              // 清空当前 namespace 所有 key（namespace 名仍保留）
prefs.freeEntries();        // 剩余条目数
prefs.getType("counter");   // 返回 PreferenceType（PT_I32/PT_U32/PT_STR/PT_BLOB/PT_INVALID ...）
prefs.end();                // 关闭 namespace
```

### 指定分区 / 只读

```cpp
prefs.begin("cfg", /*readOnly*/ true, /*partition_label*/ "nvs");   // 可省略 partition_label
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 写入失败返回 0 | namespace/key 超 15 字符，或只读模式打开 | 缩短名称；用 `false` 打开 |
| `getString` 缓冲不足 | `maxLen` 小于存储长度（含 `\0`） | 先 `getStringLength` 分配 |
| `getBytes` 返回 0 | 同上 | 先 `getBytesLength` |
| namespace 打不开 | 默认 `nvs` 分区不存在 | 分区表加 `nvs, data, nvs, 0x9000, 0x5000,` |
| `putFloat/Double` 占条目多 | float/double 占 3 个 key 表条目 | 用 `freeEntries()` 监控空间 |

## 参考

- `libraries/Preferences/examples/StartCounter/StartCounter.ino`
- 仓库文档 `docs/en/api/preferences.rst`、`docs/en/tutorials/preferences.rst`
