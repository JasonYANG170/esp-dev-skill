# ESP-Insights Real Example Index

> 仓库 `D:/esp-skill/espressif-repos/esp-insights` 下真实存在的示例与关键文件。路径相对仓库根。

## 示例工程

| 路径 | 一句话说明 |
|---|---|
| `examples/minimal_diagnostics/` | 最小集成示例：HTTPS/MQTT 二选一，采集 error/warn/event + heap/wifi metrics + network variables，是大多数新项目起点 |
| `examples/diagnostics_smoke_test/` | 端到端冒烟：随机 error/warn/event、制造 5 次崩溃（写非法地址）、malloc/free 制造 heap 曲线，用于全功能验证 |
| `unit_test_app/` | Python 测试用的单元测试应用（依赖 `tests_sequence.json`） |

## minimal_diagnostics 关键文件

| 文件 | 作用 |
|---|---|
| `examples/minimal_diagnostics/main/app_main.c` | 入口：NVS → netif/event loop → example_connect → time sync → `esp_insights_init` → 周期 dump heap/wifi metrics |
| `examples/minimal_diagnostics/main/CMakeLists.txt` | `idf_component_register` + HTTPS 时 `target_add_binary_data(... insights_auth_key.txt TEXT)` |
| `examples/minimal_diagnostics/main/idf_component.yml` | 依赖 `espressif/esp_insights >=1.1.0`（`override_path` 指向本地组件） |
| `examples/minimal_diagnostics/sdkconfig.defaults` | ENABLED + 2.0 metadata + coredump + mbedTLS 动态缓冲 + metrics/variables 全开 |
| `examples/minimal_diagnostics/sdkconfig.ci` | CI 配置：MQTT 传输 + 1.0 metadata + 调试打印 + 进阶网络变量 |
| `examples/minimal_diagnostics/partitions.csv` | nvs/phy/factory/coredump/fctry 分区 |
| `examples/minimal_diagnostics/README.md` | HTTPS 步骤 + MQTT Claiming 步骤 + core dump 配置说明 |

## diagnostics_smoke_test 关键文件

| 文件 | 作用 |
|---|---|
| `examples/diagnostics_smoke_test/main/app_main.c` | smoke_test 任务：dice 概率发 error/warn/event、5 次崩溃、malloc/free；演示跨重启日志保留与 core dump |
| `examples/diagnostics_smoke_test/main/CMakeLists.txt` | 同 minimal（含 HTTPS auth key 嵌入） |
| `examples/diagnostics_smoke_test/sdkconfig.defaults` | 与 minimal 一致（ENABLED + 2.0 + coredump + metrics/variables） |
| `examples/diagnostics_smoke_test/sdkconfig.ci` | CI 配置 |
| `examples/diagnostics_smoke_test/partitions.csv` | 分区表 |
| `examples/diagnostics_smoke_test/README.md` | 端到端验证清单（时间戳、错误计数、metrics 曲线、变量、交叉引用、崩溃、分页、访问控制） |

## 组件源码（API 权威来源）

| 组件 | 头文件 | 说明 |
|---|---|---|
| `components/esp_insights/` | `include/esp_insights.h` | agent：init/enable、transport 注册、reporting 控制、cmd-resp |
| `components/esp_diagnostics/` | `include/esp_diagnostics.h` | 日志钩子、`ESP_DIAG_EVENT`、数据类型、设备信息、任务快照 |
| `components/esp_diagnostics/` | `include/esp_diagnostics_metrics.h` | metrics 注册/上报（1.0 `add_*` / 2.0 `report_*`） |
| `components/esp_diagnostics/` | `include/esp_diagnostics_variables.h` | variables 注册/上报（1.0/2.0） |
| `components/esp_diagnostics/` | `include/esp_diagnostics_system_metrics.h` | heap metrics / wifi metrics |
| `components/esp_diagnostics/` | `include/esp_diagnostics_network_variables.h` | network variables init/deinit |
| `components/esp_diag_data_store/` | `include/esp_diag_data_store.h` | 数据存储读写/释放、低内存事件 |

## 配置来源

| 文件 | 内容 |
|---|---|
| `components/esp_insights/Kconfig` | ENABLED/transport/coredump/cmd-resp/host/上报间隔/metadata 版本 |
| `components/esp_diagnostics/Kconfig` | 日志格式/metrics/variables/网络变量/外部 wrap |
| `components/esp_diag_data_store/Kconfig` | RTC/RAM/FLASH 存储、容量、水位线 |
| `components/esp_insights/idf_component.yml` | 依赖版本（IDF >=5.1、rmaker_common、cbor 等） |

## 文档

| 文件 | 内容 |
|---|---|
| `README.md` | 概述、HTTPS/MQTT 接入、Behind the Scenes、RTC 存储 |
| `FEATURES.md` | Core dump/Logs/Reboot/Metrics/Variables/Transport Sharing/Optimisation/Group Analytics/Command Response 详解 |
| `CHANGELOG.md` | 版本历史（含 metadata 2.0 破坏性变更、IDF v6.0 兼容、Linux host 单测） |
| `docs/en/index.rst` | Sphinx 文档入口（toctree 指向各组件 Doxygen `.inc`） |
| `examples/README.md` | Getting Started（克隆、IDF 设置、账号、Auth Key） |
