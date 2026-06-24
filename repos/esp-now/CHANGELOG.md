# Changelog

## [1.1.0] - 2026-06-18

新增综合方案 recipe，补齐 `examples/solution` 这一唯一无专属 recipe 的示例目录。

### Added

- **recipes/solution_integrated_firmware.md** — 多特性集成固件：单二进制按 `CONFIG_APP_ESPNOW_INITIATOR`/`RESPONDER` 条件编译集成 Wi-Fi 配网（BLE/SoftAP，`components/wifi_prov`）+ ESP-NOW 配网 + 设备控制 + 无线调试 + 批量 OTA + 安全握手 + 节点时间同步。覆盖：
  - 角色 choice 与 `select APP_WIFI_PROVISION` 强约束（`main/Kconfig.projbuild`）。
  - `app_main` 固定初始化顺序与 `sec_enable=1`。
  - **关键约束**：responder 侧 ESP-NOW 配网 initiator 任务在 `espnow_get_key()` 上轮询，安全握手派发 app key 完成前不发加密业务数据（`components/espnow_device/responder.c`）。
  - 单按键复用映射表（`WIFI_PROV_KEY_GPIO=2` 配网 / `CONTROL_KEY_GPIO=9/0` 控制；单击/双击/长按）。
  - 共享 RGB LED 状态语义（白=配网中 / 绿=已连/绑定 / 红=断开/解绑）。
  - responder 端 sec/console/log/ota 启动代码、双固件构建流程（`PROJECT_NAME=Resp`）、`mbedtls_ccm_auth_decrypt error -15` 等常见错误。
- **SKILL.md**：新增「综合方案（产品级集成）」分组与 recipe 行；新增 Core Principle #13（集成多特性时安全握手必须先于加密业务）；`metadata.version` 1.0.0 → 1.1.0。
- **resources/example_list.md**：扩充 `examples/solution` 条目，补齐 Kconfig 选项、自定义组件清单、关键约束与配套 recipe 指引。
- **resources/api_reference.md**：新增「示例封装：examples/solution 组件」一节，列出 `app_espnow_initiator_register/_initiator/_sec_start`、`app_espnow_responder_register/_responder`、`app_espnow_prov_beacon_start/_responder_start`、`wifi_prov_init/wifi_prov` 的真实签名（取自 `components/*/include/*.h`），并标注为非组件公开 API。

### Grounding

新增内容全部取自：
- `espressif/esp-now/examples/solution/README.md`、`main/app_main.c`、`main/Kconfig.projbuild`
- `espressif/esp-now/examples/solution/components/espnow_device/{initiator.c,responder.c,include/*.h}`
- `espressif/esp-now/examples/solution/components/wifi_prov/{wifi_prov.c,include/wifi_prov.h}`
- `espressif/esp-now/User_Guide.md`

## [1.0.0] - 2026-06-18

首个发布版本。基于 `espressif/esp-now` 仓库（component v2.5.3）真实文档与源码构建。

### Added

- **SKILL.md**：技能主文件，含 12 条 Core Principles、When to Use、场景索引表、`espnow_data_type_t` 与角色参考表、13 条 Critical Pitfalls（每条含 WRONG/CORRECT 代码）、Execution Workflow（9 步 + 工程创建策略）、Failure Strategies。
- **AGENTS.md**：补充约定 — 组件依赖添加、头文件包含规范、标准工程结构、`app_main` 标准模式、错误处理宏、内存宏、构建工作流、代码生成 checklist、Do Not Modify 说明。
- **recipes/（11 个）**：
  - `get_started_send_recv.md` — 入门收发（get-started）
  - `unicast_and_group.md` — 单播 peer 与分��
  - `control_initiator.md` — 控制发起方（control）
  - `control_responder.md` — 控制响应方（control / bulb）
  - `coin_cell_switch.md` — 硬币电池低功耗开关
  - `security_handshake.md` — 安全握手与加解密（security）
  - `ota_batch_upgrade.md` — 批量 OTA（ota）
  - `provisioning_wifi.md` — Wi-Fi 配网（provisioning）
  - `wireless_debug.md` — 无线调试日志/console/命令
  - `time_sync.md` — 节点间时间同步
  - `storage_utils.md` — storage / mem / utils 工具
- **resources/api_reference.md**：按模块汇总全部公开 API 签名（espnow / ctrl / security / handshake / ota / prov / time / utils / debug）。
- **resources/config_reference.md**：列出仓库 `Kconfig` 全部选项（安全、light sleep、control、OTA、任务、utils、debug）及 IDF 版本兼容表。
- **resources/pitfalls.md**：集中 39 条常见陷阱与修正，分初始化/收发/安全/OTA/控制/配网/低功耗/内存/构建。
- **resources/example_list.md**：9 个官方示例索引与 `idf.py create-project-from-example` 下载命令。
- **README.md**：中文介绍、特性、适用范围、安装、目录结构。
- **CHANGELOG.md**：本文件。

### Grounding

所有函数名、结构体、宏、Kconfig 选项、示例路径与代码片段均取自：
- `espressif/esp-now/src/*/include/*.h`（API 权威来源）
- `espressif/esp-now/examples/*/main/app_main.c`（真实示例代码）
- `espressif/esp-now/Kconfig`、`idf_component.yml`、`User_Guide.md`、`User_Guide_CN.md`

无臆造内容；仓库文档较薄处（如 wireless_debug 的 monitor 封装）明确标注为示例内部而非公开 API。
