# 陷阱汇总（Pitfalls）

> 汇总 ESP Zigbee SDK v2.x 开发中最常见的错误与对策。每条均给出原因与解决方法。SKILL.md 中有带代码的"错误/正确"对照，本篇为检索版。

## 一、API 与线程

1. **任何 `ezb_` 调用前未加锁** → 除 Zigbee 主循环回调内、`esp_zigbee_task_queue_post` 投递回调内两种情形外，必须 `esp_zigbee_lock_acquire/release` 包裹。
2. **v1.x `esp_zb_` 与 v2.x `ezb_` 混用** → 新代码统一用 `ezb_`；老代码迁移看 `docs/en/migration-guide/v2.x/`，或临时用 `compat/` 兼容层。
3. **主任务栈不足** → 统一 4096；用 `uxTaskGetStackHighWaterMark` 监控。
4. **在 ZCL core action 回调里阻塞** → 该回调在主任务上下文，阻塞会卡死协议栈；耗时工作用 `esp_zigbee_task_queue_post` 投递。

## 二、启动流程

5. **主任务缺 `esp_zigbee_launch_mainloop()`** → 协议栈不运转。
6. **init/start/mainloop 顺序错** → 必须 `esp_zigbee_init` → 配置/数据模型 → `esp_zigbee_start(false)` → `esp_zigbee_launch_mainloop`。
7. **漏处理 `EZB_ZDO_SIGNAL_SKIP_STARTUP`** → `ezb_bdb_start_top_level_commissioning(EZB_BDB_MODE_INITIALIZATION)` 不会被触发，`FIRST_START` 永不到来。
8. **commissioning 失败不重试** → 用 `alarm_timer_schedule(cb, MODE, 1000)` 延迟 1s 重试（示例统一模式）。

## 三、角色与 BDB

9. **ZC 用 steering 而非 formation** → factory-new ZC 须先 `EZB_BDB_MODE_NETWORK_FORMATION`，成功后再 `STEERING`；非 factory-new 用 `ezb_bdb_open_network`。
10. **ZED 用 formation** → `EZB_BDB_STATUS_NOT_PERMITTED`；ZR/ZED 入网只能 `STEERING`。
11. **device_type 与角色不符** → switch 用 `END_DEVICE`/`ROUTER`，灯/网关用 `COORDINATOR`/`ROUTER`。
12. **想开放网络但时长短** → `ezb_bdb_open_network(seconds)` 给足够秒数（示例 180s）。

## 四、数据模型

13. **endpoint 创建后未 register** → 必须四步：`create_device_desc` → `create_<device>` → `add_endpoint_desc` → `device_desc_register`。
14. **ManufacturerName 乱码** → Zigbee 字符串首字节是长度，写 `"\x09""ESPRESSIF"`。
15. **Basic 属性加不上** → 取 cluster_desc 时角色须为 `EZB_ZCL_CLUSTER_SERVER`。
16. **多 endpoint 漏 register** → 多个 ep 都 `add_endpoint_desc` 后只调一次 `device_desc_register`。

## 五、属性与命令

17. **`EZB_ADDR_MODE_NONE` 上报无效** → 须先 ZDO bind，由绑定表路由；否则用 `EZB_ADDR_MODE_SHORT/GROUP` 显式寻址。
18. **report 方向错** → client→server 用 `EZB_ZCL_CMD_DIRECTION_TO_CLI`。
19. **温度单位错** → 必须 `int16_t`（×100，0.01℃/单位）。
20. **level 不变** → 灯未开时用 `move_to_level_with_on_off`。
21. **颜色命令编译错** → 以 `color_control.h` 实际函数名为准，不要凭记忆。

## 六、存储与分区

22. **缺 `zb_storage` / `zb_fct` 分区** → `partitions.csv` 必须含（16K nvs + 1K fat）。
23. **未初始化专用 NVS** → `app_main` 必须 `nvs_flash_init_partition(ESP_ZIGBEE_STORAGE_PARTITION_NAME)`。
24. **首次烧录未 erase_flash** → 旧持久化数据可能导致行为异常，首次 `idf.py erase_flash flash`。

## 七、休眠与网关

25. **休眠终端功耗高** → `ezb_nwk_set_rx_on_when_idle(false)` + `CONFIG_PM_ENABLE=y` + `CONFIG_FREERTOS_USE_TICKLESS_IDLE=y`。
26. **终端掉线** → `keep_alive=4000`，`ed_timeout` 与父节点协商（≥ `EZB_NWK_ED_TIMEOUT_64MIN`）。
27. **非 15.4 SoC 用 NATIVE** → C3/S3/P4 必须 `ESP_ZIGBEE_RADIO_MODE_UART_RCP` + UART 引脚 + 460800 波特。
28. **RCP 不通信** → 接线 `rx_pin=主控收=RCP_TX`，RCP 已烧 `ot_rcp`。
29. **共存 Wi-Fi 不稳** → `esp_wifi_set_ps(WIFI_PS_MIN_MODEM)` + `esp_coex_wifi_i154_enable()`。
30. **C6 单芯片网关选 RCP 模式** → C6 有 15.4，单芯片用 `NATIVE` 即可。

## 八、调试

31. **抓包看不到 payload** → Wireshark 添加预配置 key `ZigbeeAlliance09`，802.15.4 与 Zigbee 协议偏好按 `developing.rst` 设置。
32. **库内断言难定位** → 收集完整串口日志 + `build/*.elf`，提 GitHub issue。
33. **找不到 `esp_zigbee.h`** → `idf_component.yml` 漏声明 `espressif/esp-zigbee-lib`，`idf.py reconfigure`。
34. **编译报版本不匹配** → 删 `build/`、`sdkconfig`、`sdkconfig.old`、`dependencies.lock` 重新构建；确认 IDF/SDK 分支正确（见 `docs/en/faq.rst`）。
