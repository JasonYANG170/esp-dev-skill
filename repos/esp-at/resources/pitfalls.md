# ESP-AT 常见陷阱汇总

> 与 `SKILL.md` 的 Critical Pitfalls 互补，本文件按类别归类实际开发中最常踩的坑，均来自仓库文档与头文件。

## 一、自定义指令

### 1. 忘记 `ESP_AT_CMD_SET_INIT_FN`

只实现 `esp_at_custom_cmd_register()` 而不加初始化宏，链接器可能丢弃该函数，指令不生效。必须加 `ESP_AT_CMD_SET_INIT_FN(fn, priority)`，且 `CMakeLists.txt` 设 `idf_component_set_property(${COMPONENT_NAME} WHOLE_ARCHIVE TRUE)`，必要时加 `target_link_libraries(${COMPONENT_LIB} INTERFACE "-u esp_at_custom_cmd_register")`。

### 2. 用了已 deprecated 的 `_RESULT_` 枚举名

`esp_at_get_para_as_digit/str` 返回 `ESP_AT_PARA_PARSE_RET_OK/_FAIL/_OMITTED`。旧名 `ESP_AT_PARA_PARSE_RESULT_*` 仍可编译（esp_at_legacy.h 提供别名）但会触发 deprecation 警告，新代码必须用 `_RET_`。

### 3. 可选参数未判断 `_OMITTED`

把 `ESP_AT_PARA_PARSE_RET_OMITTED` 当 `FAIL` 处理，导致可选参数报错。正确做法：`FAIL` 报错，`OMITTED` 用默认值，`OK` 用解析值。注意空串 `""` 不等于省略。

### 4. 在 handler 内调用 `esp_at_exe_cmd`

`esp_at_exe_cmd`（需 `CONFIG_AT_SELF_COMMAND_SUPPORT`）会阻塞等待响应，在 test/query/setup/exe handler 内直接调用会死锁。应通过独立任务+队列派发。

### 5. 指令结束后端口卡住

`esp_at_write_result()` 仅输出结果串不改端口状态；指令完全结束、需要接收新数据时应使用 `esp_at_dispatch_result(ESP_AT_RESULT_CODE_OK, NULL)` 恢复 ready 态。

### 6. 指令命名非法

必须以 `+` 开头；允许字符仅 `A-Za-z0-9` 及 `! % - . / : _`。其它字符会被 AT 引擎拒绝。

## 二、构建与环境

### 7. 未设置 `AT_CUSTOM_COMPONENTS`

自定义组件（at_custom_cmd / at_override_module_config）必须通过环境变量 `AT_CUSTOM_COMPONENTS` 指向其目录，否则不会被纳入构建。多组件用空格分隔。

### 8. `module_info.json` 缓存导致无法重选模块

`build.py install` 后 `build/module_info.json` 缓存 Platform/Module 选择。需要重新配置时删除该文件再 install。

### 9. 路径含空格

ESP-AT 不支持路径含空格。Windows 下避免放在 `Program Files` 等目录。

### 10. `ota data partition invalid` 启动失败

烧录后报此错。解决：`./build.py erase_flash` 后重新 `flash`。

### 11. silence mode 误开

silence mode 会裁剪日志、减小固件体积，调试时不便。一般 install 时选 No（`0`）。

## 三、引脚与端口

### 12. 改错文件改引脚

引脚定义在 `factory_param_data.csv` 的 `uart_tx_pin/uart_rx_pin/uart_cts_pin/uart_rts_pin` 列。不要改 `partitions_at.csv`（系统主分区表，改了可能无法启动）。

### 13. 命令引脚与日志引脚冲突

两者不能占用同一组引脚。日志引脚在 menuconfig（`ESP System Settings` → console UART）改，命令引脚在 CSV 改。

### 14. 不用流控时 CTS/RTS 置 -1

不使用硬件流控时，CSV 中 `uart_cts_pin`、`uart_rts_pin` 设 `-1`，否则可能占用其它功能引脚。

## 四、分区与存储

### 15. 未烧 `at_customize.bin`

`AT+SYSFLASH`、`AT+FS`、SSL 服务端、BLE 服务端功能依赖 `at_customize.bin` 已烧到正确地址。只烧 factory/ota bin 会导致这些指令不可用。

### 16. 用户分区 Name 超长

`at_customize.csv` 中用户分区 `Name` ≤ 16 字节。

### 17. 新增分区 Type 误用

ESP-IDF `esp_partition.h` 已定义的 Type 需沿用；自定义分区 Type 用 `0x40`。

### 18. 用户分区超范围

新增分区不能超出 `partitions_at.csv` 定义的 `at_customize` 整体范围。

## 五、功能与性能

### 19. HTTPS 任务栈过小崩溃

开启 `AT_HTTP_COMMAND_SUPPORT` 后，建立 HTTPS 链路需更大栈。务必把 `AT_PROCESS_TASK_STACK_SIZE` 调到 4096 以上。

### 20. 蓝牙使固件体积超 OTA 分区

开启蓝牙等功能后固件显著变大，可能超 OTA 分区。需扩大 OTA 分区或裁剪非必要功能。

### 21. ESP32-C2 默认无 BLE

ESP32-C2 的 `AT_BLE_COMMAND_SUPPORT` 默认为 `n`（其它芯片默认 y）。C2 启用 BLE 需手动配置且依赖 NimBLE。

### 22. Classic BT 仅 ESP32

`AT_BT_COMMAND_SUPPORT` 仅 ESP32 可用。ESP32-S3/H2 不被 ESP-AT 支持。

## 六、OTA

### 23. OTA 用户分区无备份

ESP-AT 把新固件存到备用 OTA 分区，OTA 失败原固件仍可运行；但**用户分区无备份**，升级用户分区需谨慎。

### 24. 刷非官方固件后丢 `AT+CIUPDATE`

升级非官方固件后可能无法再用 `AT+CIUPDATE`（除非在 iot.espressif.cn 建设备 + 自有 token）。若计划后续用自定义固件 + `AT+CIUPDATE`，初始版本就应配自有 OTA token。

### 25. OTA bin 命名错误

升级 app：bin 名必须为 `ota.bin`；升级用户分区：bin 名为 `at_customize.csv` 中分区 `Name` + `.bin`。

## 七、BLE 服务

### 26. gatts_data.csv perm 字段语义混淆

`gatts_data.csv` 第二行（UUID `0x2803` 特征声明）的 perm 是**特征属性位**（READ/NOTIFY/WRITE 等），不是访问权限。访问权限用 `ESP_GATT_PERM_*`（0x0001/0x0010...）。

### 27. UUID 长度不匹配

`uuid_len` 必须与 `uuid` 字段实际字节数一致（通常 16）。
