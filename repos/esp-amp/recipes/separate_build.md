# Separate Build：分别构建两侧固件

> **适用摘要**: 使用 ESP-AMP separate build 模式，maincore 与 subcore 各自独立工程、独立构建。适合 subcore 固件需独立开发或由第三方提供的场景。subcore 固件必须手动 esptool 烧入 flash 分区。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-amp/resources/`, source/examples in `repos/esp-amp/`, and this recipe path `repos/esp-amp/recipes/separate_build.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "separate build"
- "分别构建 maincore 和 subcore"
- "subcore 独立工程"
- "subcore 固件第三方开发"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `examples/build_system/separate_build/` |
| ESP-IDF | v5.3.1+（C6/P4）或 v5.5+（C5） |
| 一致性 | maincore 与 subcore 的 ESP-AMP 相关 sdkconfig（共享内存大小、SysInfo/Event）必须手动保持一致 |

## 分步说明

### 1. 工程目录结构

```
my_amp_separate/
├── common/                         # 双核共享头（event.h / sys_info.h）
├── maincore_project/
│   ├── CMakeLists.txt
│   ├── partitions.csv
│   ├── sdkconfig.defaults
│   └── main/
│       ├── CMakeLists.txt
│       └── app_main.c
└── subcore_project/
    ├── CMakeLists.txt
    ├── sdkconfig.defaults          # 注意：ESP-AMP 相关项须与 maincore 一致
    └── main/
        ├── CMakeLists.txt
        └── main.c
```

### 2. maincore_project/partitions.csv

```csv
# Name,      Type,       SubType,    Offset,      Size
nvs,         data,       nvs,        0x9000,      24K,
phy_init,    data,       phy,        0xf000,      4K,
factory,     app,        factory,    0x10000,     1M,
sub_core,    data,       0x40,       0x200000,    16K,
```

### 3. 构建并烧录 maincore

```shell
cd maincore_project
idf.py set-target esp32c6
idf.py build
idf.py flash
```

### 4. 构建 subcore

```shell
cd ../subcore_project
idf.py set-target esp32c6
idf.py build
```
- 生成的 subcore 固件：`subcore_project/build/subcore_<name>.bin`

### 5. 手动烧录 subcore 到分区

```shell
# offset 必须等于 partitions.csv 中 sub_core 条目的 Offset
esptool.py write_flash 0x200000 build/subcore_separate_build.bin
```

### 6. maincore 加载代码（与 unified build 分区模式相同）

```c
const esp_partition_t *sub_partition = esp_partition_find_first(ESP_PARTITION_TYPE_DATA, 0x40, NULL);
ESP_ERROR_CHECK(esp_amp_load_sub_from_partition(sub_partition));
ESP_ERROR_CHECK(esp_amp_start_subcore());
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| subcore 加载失败 / 行为异常 | 两侧 `CONFIG_ESP_AMP_HP_SHARED_MEM_SIZE` 等不一致 | separate build 不共享 sdkconfig，必须手动核对 ESP-AMP 相关项一致 |
| esptool 烧录地址错误 | offset 与 `partitions.csv` 不符 | offset 取 `partitions.csv` 中 sub_core 条目的 Offset 列 |
| subcore 固件找不到分区 | type/subtype 与 `esp_partition_find_first` 参数不匹配 | `esp_partition_find_first(ESP_PARTITION_TYPE_DATA, 0x40, NULL)` 与分区表一致 |
| subcore 不支持嵌入 | separate build 不支持 EMBED | 嵌入必须改用 unified build |
| separate build 下 maincore 仍报 subcore 编译错误 | 误把 subcore 路径加入 maincore 的 `EXTRA_COMPONENT_DIRS` | separate build 两侧完全独立，不要交叉引用 |

## 参考

- `examples/build_system/separate_build/` — separate build 完整示例（含手动 esptool 烧录步骤）
- `espressif-repos/esp-amp/docs/build_system.md`
