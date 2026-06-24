# ESP RainMaker 配置（Kconfig）速查

> 全部来自 `components/esp_rainmaker/Kconfig.projbuild`。menu 路径 “ESP RainMaker Config��。

## Claiming / PKI

| 符号 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_ESP_RMAKER_CLAIM_TYPE` | choice | — | 0=No Claim, 1=Self, 2=Assisted |
| `CONFIG_ESP_RMAKER_NO_CLAIM` | bool | — | 不用 claiming，MQTT 凭据需预烧（私有部署） |
| `CONFIG_ESP_RMAKER_SELF_CLAIM` | bool | — | 自助 claiming；**不支持 ESP32/ESP32-C2** |
| `CONFIG_ESP_RMAKER_ASSISTED_CLAIM` | bool | BT 且非 S2 时默认 | 辅助 claiming，需蓝牙 |
| `CONFIG_ESP_RMAKER_CLAIM_KEY_RSA` | bool | — | RSA 2048 密钥（向后兼容） |
| `CONFIG_ESP_RMAKER_CLAIM_KEY_ECDSA` | bool | 默认 | ECDSA P-256 密钥（推荐） |
| `CONFIG_ESP_RMAKER_USE_NVS` | bool | 默认 | PKI 凭据存 NVS |
| `CONFIG_ESP_RMAKER_USE_ESP_SECURE_CERT_MGR` | bool | — | 用 ESP Secure Cert Manager（仅 No Claim） |
| `CONFIG_ESP_RMAKER_CLAIM_SERVICE_BASE_URL` | string | `https://esp-claiming.rainmaker.espressif.com` | claiming 服务地址（Self Claim） |
| `CONFIG_ESP_RMAKER_CLAIM_VIDEOSTREAM_SUPPORT` | bool | n | 摄像头视频流 claiming（KVS） |

## MQTT

| 符号 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_ESP_RMAKER_MQTT_HOST` | string | `mqtt.rainmaker.espressif.com` | MQTT host（claim/覆盖时） |
| `CONFIG_ESP_RMAKER_MQTT_CRED_HOST` | string | AWS 凭据端点 | 凭据端点（不同 node_policies�� |
| `CONFIG_ESP_RMAKER_READ_MQTT_HOST_FROM_CONFIG` | bool | n | 强制用 `MQTT_HOST` 覆盖 NVS |
| `CONFIG_ESP_RMAKER_READ_NODE_ID_FROM_CERT_CN` | bool | n | 从证书 CN 读 node id |
| `CONFIG_ESP_RMAKER_MQTT_USE_BASIC_INGEST_TOPICS` | bool | y | AWS Basic Ingest 主题（降成本） |
| `CONFIG_ESP_RMAKER_MQTT_ENABLE_BUDGETING` | bool | y | 启用 MQTT 预算限流 |
| `CONFIG_ESP_RMAKER_MQTT_DEFAULT_BUDGET` | int | 100 (64~MAX) | 初始预算 |
| `CONFIG_ESP_RMAKER_MQTT_MAX_BUDGET` | int | 1024 (64~2048) | 预算上限 |
| `CONFIG_ESP_RMAKER_MQTT_BUDGET_REVIVE_PERIOD` | int | 5 (5~600) | 预算恢复周期（秒） |
| `CONFIG_ESP_RMAKER_MQTT_BUDGET_REVIVE_COUNT` | int | 1 (1~16) | 每周期恢复数 |
| `CONFIG_ESP_RMAKER_MAX_PARAM_DATA_SIZE` | int | 1024 (64~8192) | 参数上报 payload 最大字节 |
| `CONFIG_ESP_RMAKER_USE_CERT_BUNDLE` | bool | y | 用证书 bundle（select `_MQTT_USE_CERT_BUNDLE`） |

## 安全 / 用户映射

| 符号 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_ESP_RMAKER_DISABLE_USER_MAPPING_PROV` | bool | n | 禁用配网期 user mapping（避免重复注册） |
| `CONFIG_ESP_RMAKER_USER_ID_CHECK` | bool | n | user-node mapping 时校验 user id（依赖 `FACTORY_RESET_REPORTING`） |
| `CONFIG_ESP_RMAKER_FACTORY_RESET_REPORTING` | bool | n | factory reset 上报云端清理旧映射 |
| `CONFIG_ESP_RMAKER_ENABLE_CHALLENGE_RESPONSE` | bool | y | 配网挑战-响应（取代传统 user mapping；需已 claim） |
| `CONFIG_ESP_RMAKER_ON_NETWORK_CHAL_RESP_ENABLE` | bool | n | on-network HTTP chal_resp（与 LOCAL_CTRL 互斥） |
| `CONFIG_ESP_RMAKER_ENABLE_PROV_LOCAL_CTRL` | bool | n | 配网期 get_params/set_params/get_config 端点（BLE-only 设备） |
| `CONFIG_RMAKER_NAME_PARAM_CB` | bool | n | 让应用回调处理 Name 参数 |

## 本地控制

| 符号 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_ESP_RMAKER_LOCAL_CTRL_FEATURE_ENABLE` | bool | n | 本地控制功能（select `ESP_HTTPS_SERVER_ENABLE`） |
| `CONFIG_ESP_RMAKER_LOCAL_CTRL_AUTO_ENABLE` | bool | n | RainMaker 启动时自动启用 |
| `CONFIG_ESP_RMAKER_LOCAL_CTRL_HTTP_PORT` | int | 8080 | HTTP 端口 |
| `CONFIG_ESP_RMAKER_LOCAL_CTRL_STACK_SIZE` | int | 6144 | HTTP server 任务栈 |
| `CONFIG_ESP_RMAKER_LOCAL_CTRL_SECURITY_0/1/2` | choice | sec1 | 安全等级 |
| `CONFIG_ESP_RMAKER_LOCAL_CTRL_CHAL_RESP_ENABLE` | bool | n | 本地控制内 chal_resp 端点 |
| `CONFIG_ESP_RMAKER_LOCAL_CTRL_LEASE_INTERVAL_SECONDS` | int | 180 | Thread BR SRP 服务租约（秒） |

## OTA（“ESP RainMaker OTA Config”）

| 符号 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_ESP_RMAKER_OTA_AUTOFETCH` | bool | y | 连接后主动拉取 OTA |
| `CONFIG_ESP_RMAKER_OTA_AUTOFETCH_PERIOD` | int | 0 (0~168) | 周期拉取（小时）；0=仅首次 |
| `CONFIG_ESP_RMAKER_SKIP_COMMON_NAME_CHECK` | bool | n | 跳过证书 CN 校验 |
| `CONFIG_ESP_RMAKER_SKIP_VERSION_CHECK` | bool | n | 跳过版本检查（仅开发） |
| `CONFIG_ESP_RMAKER_SKIP_SECURE_VERSION_CHECK` | bool | n | 跳过 secure version（anti-rollback）检查 |
| `CONFIG_ESP_RMAKER_SKIP_PROJECT_NAME_CHECK` | bool | n | 跳过工程名检查 |
| `CONFIG_ESP_RMAKER_OTA_ROLLBACK_WAIT_PERIOD` | int | 90 (30~600) | 回滚等待（秒） |
| `CONFIG_ESP_RMAKER_OTA_ROLLBACK_REPORT_FAILED` | bool | n | MQTT 超时回滚时报 failed |
| `CONFIG_ESP_RMAKER_OTA_TEST_ROLLBACK` | bool | n | 测试用：成功也回滚 |
| `CONFIG_ESP_RMAKER_OTA_TEST_ROLLBACK_COUNT` | int | 100 (0~100000) | 测试回滚次数 |
| `CONFIG_ESP_RMAKER_OTA_DISABLE_AUTO_REBOOT` | bool | n | OTA 后不自动重启 |
| `CONFIG_ESP_RMAKER_OTA_TIME_SUPPORT` | bool | y | OTA 时间窗支持 |
| `CONFIG_ESP_RMAKER_OTA_PROGRESS_SUPPORT` | bool | n | 上报下载进度 |
| `CONFIG_ESP_RMAKER_OTA_PROGRESS_INTERVAL` | int | 10 (5~50) | 进度间隔（%） |
| `CONFIG_ESP_RMAKER_OTA_MAX_RETRIES` | int | 3 (1~10) | 最大重试 |
| `CONFIG_ESP_RMAKER_OTA_RETRY_DELAY_MINUTES` | int | 5 (1~60) | 重试延迟（分钟） |
| `CONFIG_ESP_RMAKER_HTTP_OTA_RESUMPTION` | bool | y | HTTP OTA 断点续传 |
| `CONFIG_ESP_RMAKER_OTA_USE_HTTPS` / `_USE_MQTT` | choice | HTTPS | OTA 协议 |
| `CONFIG_ESP_RMAKER_OTA_HTTP_RX_BUFFER_SIZE` | int | 1024 | HTTPS OTA 接收缓冲 |
| `CONFIG_ESP_RMAKER_MQTT_OTA_BLOCK_SIZE` | int | 4096 (256~131072) | MQTT OTA 块大小 |
| `CONFIG_ESP_RMAKER_MQTT_OTA_NO_OF_BLOCKS` | int | 16 (1~512) | MQTT OTA 块数（块×大小≤128KB） |
| `CONFIG_ESP_RMAKER_MQTT_OTA_MAX_RETRIES` | int | 3 (1~255) | MQTT OTA 重试 |
| `CONFIG_ESP_RMAKER_MQTT_OTA_BLOCK_WAIT_SEC` | int | 10 (5~300) | MQTT OTA 单块等待（秒） |

## 时间 / 调度 / 场景 / 连接性

| 符号 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_ESP_RMAKER_TIME_CURRENT_TIME_PARAM` | bool | n | Time 服务加 Current-Time 参数 |
| `CONFIG_ESP_RMAKER_SCHEDULING_MAX_SCHEDULES` | int | 10 (1~50) | 最大调度数 |
| `CONFIG_ESP_RMAKER_SCHEDULE_ENABLE_DAYLIGHT` | bool | y | 日光调度（依赖 `ESP_SCHEDULE_ENABLE_DAYLIGHT`） |
| `CONFIG_ESP_RMAKER_SCENES_MAX_SCENES` | int | 10 (1~50) | 最大场景数 |
| `CONFIG_ESP_RMAKER_SCENES_DEACTIVATE_SUPPORT` | bool | n | 场景反激活回调 |
| `CONFIG_ESP_RMAKER_CONNECTIVITY_REPORT_DELAY` | int | 5 (0~30) | 上报 Connected=true 延迟（秒） |

## 命令-响应（Command-Response）

| 符号 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_ESP_RMAKER_CMD_RESP_ENABLE` | bool | y | 命令-响应模块 |
| `CONFIG_ESP_RMAKER_PARAM_CMD_RESP_ENABLE` | bool | y | 参数 set 自动注册到 cmd_resp |
| `CONFIG_ESP_RMAKER_PARAM_CMD_RESP_BUFFER_SIZE` | int | 2048 (512~8192) | 参数 cmd_resp 缓冲 |
| `CONFIG_ESP_RMAKER_CMD_RESP_TEST_ENABLE` | bool | n | 测试：节点自发自收命令 |

## 控制台

| 符号 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_ESP_RMAKER_CONSOLE_UART_NUM_0/1` | choice | UART0 | 控制台 UART |
| `CONFIG_ESP_RMAKER_CONSOLE_PARAM_CMDS_ENABLE` | bool | n | set-param/update-param/get-param 命令 |
| `CONFIG_ESP_RMAKER_CONSOLE_CHAL_RESP_CMDS_ENABLE` | bool | n | chal-resp-enable/disable 命令 |

## 网络

| 符号 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_ESP_RMAKER_NETWORK_OVER_WIFI` | choice | 默认 | RainMaker over Wi-Fi（依赖 wifi enabled） |
| `CONFIG_ESP_RMAKER_NETWORK_OVER_THREAD` | choice | — | RainMaker over Thread（依赖 `OPENTHREAD_ENABLED`） |

> 另有 `components/esp_rainmaker/sdkconfig.rename` 用于历史符号重命名映射，迁移老工程时参考。
