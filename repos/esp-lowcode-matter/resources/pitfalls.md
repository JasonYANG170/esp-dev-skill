# 陷阱汇总（esp-lowcode-matter）

> 与 `SKILL.md` 的 Critical Pitfalls 互补，本文件按主题归并，便于快速查阅。所有内容均来自仓库源码与 `docs/debugging.md`、`docs/programmer_model.md`、`docs/product_configuration.md`。

## 一、骨架与生命周期

1. **`system_setup()` 必须是 `main()` 第一条应用语句**，且永不可省略——否则 LP Core 系统未初始化。
2. **`while(1)` 必须同时跑 `system_loop()` 和 `loop()`**；`loop()` 内必须调用 `low_code_get_feature_update_from_system()` 与 `low_code_get_event_from_system()` 两个拉取函数。
3. **回调注册早于驱动初始化**：`setup()` 内先 `low_code_register_callbacks(...)` 再 `app_driver_init()`，否则初始化期间的事件/特性丢失。

## 二、��性（feature）路由

4. **`feature_update_from_system()` 必须按 `endpoint_id` + `feature_id` 双重判断分发**，否则多 endpoint/多 feature 产品会误动作（见 `socket_2_channel`、`light_rgbcw_ws2812`）。
5. **上报时 `value.type` 必须与数据类型一致**：温度用 `INTEGER`（带符号，°C*100），通断/占用用 `BOOLEAN`/`UNSIGNED_INTEGER`，不要混用。
6. **无 `feature_id` 映射时必须填 `low_level.matter.cluster_id/attribute_id`**，否则系统侧无法路由。
7. **`value` 指针在 `low_code_feature_update_to_system()` 调用期间必须有效**；避免上报已出作用域的局部变量。

## 三、事件（event）

8. **工厂复位是上报 `LOW_CODE_EVENT_FACTORY_RESET`**，不要在 LP Core 自己擦 Matter 凭证分区。
9. **`app_driver_event_handler()` 应 switch 覆盖所需事件**，并给出灯效/显示指示；至少处理 SETUP/NETWORK/OTA/READY/IDENTIFICATION 系列。
10. **测试模式 subtype 从 `event_data` 读取**：`int subtype = *((int*)(event->event_data));`，不要当 `event_type` 用。

## 四、外设/驱动

11. **WS2812 用 `ws2812_io`，PWM LED 用 `led_io`**——`io_conf` 是 union，混用成员无效。
12. **Matter↔driver 量程换算**：亮度 Matter 0-255 → driver 0-100（`brightness*100/255`）；色温 mireds ↔ Kelvin（`1000000/temperature`）。
13. **LP Core GPIO 只能用 `system_*` API**（`system_set_pin_mode/digital_write/digital_read`），不要用标准 ESP-IDF `gpio_*`。
14. **按键需软件中断**：HP GPIO 按键依赖 `system_enable_software_interrupt()`（button 组件内部使用），Kconfig 选 HP/LP 要与硬件一致。
15. **按键长按复位用 `BUTTON_LONG_PRESS_UP`**，不要用 `BUTTON_PRESS_DOWN`，避免误触。
16. **SSD1306 绘制后必须 `display_ssd1306_refresh_gram()`**，且先 `clear_screen(0x00)`；地址默认 `0x3C`。SSD1315 由该驱动通用支持。
17. **传感器周期上报回调签名固定为 `(system_timer_handle_t, void*)`**——未用参数也要保留。

## 五、内存与稳定性（`docs/debugging.md`）

18. **LP Core 禁用 `malloc/calloc`**，改静态缓冲；禁止深递归（改迭代），恒定栈占用。
19. **显式定义并严守缓冲边界**，数组/缓冲绝不越界。
20. **非法内存访问不一定 panic**：读非法内存**总是返回 0x0**，写可能静默——因此日志比依赖崩溃更重要。
21. **Breakpoint panic** 用 `riscv32-esp-elf-addr2line -e <elf> <MEPC>` 定位（空指针）。
22. **Illegal Instruction panic**（寄存器全 0）多为缓冲/栈溢出，栈转储不可靠，转用日志逐步定位。

## 六、构建与配置

23. **新增组件必须在 `main/CMakeLists.txt` 的 `REQUIRES` 声明**，否则符号未定义。
24. **改 `data_model.zap` / `product_info.json` / `product_config.json` 后必须重跑「Upload Configuration」**，重新生成并烧录 `data_model.bin` 与证书，否则设备行为不变。
25. **数据模型合规**：endpoint 0 必须为 root node；新增 endpoint/cluster/attribute 会增加内存占用，需测试。
26. **`chip` 必须为 `esp32c6`**，`connection_type`（wifi/thread）须与对应 `data_model_*.zap` 匹配。
27. **Prepare Device（烧 HP 预编译镜像）与 Upload Configuration 每块板各只做一次**；之后改代码只重跑 Upload Code（烧 `build/<product>.bin` @ `0x20C000`）。

## 七、范围限制

28. **只在 LP Core 写产品代码**；HP Core 跑 `pre_built_binaries/` 预编译镜像，不要尝试直接调用 Wi-Fi/BLE/Matter 栈 API。
29. **框架当前仅支持 ESP32-C6**；其它芯片不支持。
30. **`analogWrite`/`analogRead` 仓库标注 TODO 未实现**——PWM 经 `light_driver`，LP ADC 未暴露。
