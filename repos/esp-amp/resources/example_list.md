# ESP-AMP 真实 example 索引

> 所有路径相对 `espressif-repos/esp-amp/examples/`。支持目标列来自各 example 的 `README.md`。

| example 路径 | 一句话描述 | 支持目标 |
|---|---|---|
| `examples/build_system/unified_build/` | Unified build：单命令同时构建 maincore + subcore，支持嵌入 `.rodata` 或烧入分区 | C5/C6/P4 |
| `examples/build_system/separate_build/` | Separate build：maincore 与 subcore 独立工程，手动 esptool 烧录 subcore 到分区 | C5/C6/P4 |
| `examples/event/` | Event 同步：创建事件、绑定 EventGroup、跨核通知与等待/轮询 | C5/C6/P4 |
| `examples/software_interrupt/` | 软件中断：注册多个 handler 到同一中断源、触发对端核 | C5/C6/P4 |
| `examples/virtqueue/` | Virtqueue：subcore(master)→maincore(remote) 单向数据流 | C5/C6/P4 |
| `examples/rpmsg_send_recv/` | RPMsg：双核互发消息，含中断/轮询双模式、多端点、零拷贝 | C5/C6/P4 |
| `examples/rpc/maincore_client_subcore_server/` | RPC：maincore FreeRTOS client + subcore bare-metal server | C5/C6/P4 |
| `examples/rpc/subcore_client_maincore_server/` | RPC：subcore bare-metal client + maincore FreeRTOS server | C5/C6/P4 |
| `examples/light_sleep/` | LP subcore 自动 light sleep：PM 配置、linker fragment 内存放置、skip/resume 宏 | C5/C6 |

## 每个 example 的关键文件

大多数 example（unified build 风格）遵循统一结构：

```
<example>/
├── CMakeLists.txt                  # 顶层（maincore）工程
├── partitions.csv                  # 含 sub_core (data, 0x40) 条目
├── sdkconfig.defaults              # 顶层共享 sdkconfig
├── sdkconfig.defaults.<target>     # 按目标覆盖（esp32c5/c6/p4）
├── common/                         # 跨核共享头（event.h / sys_info.h / rpc_service.h）
├── maincore/
│   ├── CMakeLists.txt              # 调用 esp_amp_add_subcore_project()
│   └── main/app_main.c             # maincore 入口（app_main）
└── subcore/
    ├── CMakeLists.txt              # subcore 工程
    ├── subcore_config.cmake        # SUBCORE_APP_NAME / SUBCORE_PROJECT_DIR
    ├── main/
    │   ├── CMakeLists.txt
    │   └── main.c                  # subcore 入口（main，bare-metal）
    └── components/sub_main_cfg/    # 仅为生成 subcore main 的 Kconfig
```

> separate_build example 结构不同：`maincore_project/` + `subcore_project/` 两个独立工程。

## 复制模板建议

| 用户需求 | 推荐模板 |
|---|---|
| 入门 / 通用 IPC | `examples/rpmsg_send_recv/`（结构最完整） |
| Event 同步 | `examples/event/` |
| 软件中断 | `examples/software_interrupt/` |
| Virtqueue 直接使用 | `examples/virtqueue/` |
| RPC | `examples/rpc/maincore_client_subcore_server/` 或反向 |
| 构建方式参考 | `examples/build_system/unified_build/` 或 `separate_build/` |
| 低功耗（仅 C5/C6） | `examples/light_sleep/` |
