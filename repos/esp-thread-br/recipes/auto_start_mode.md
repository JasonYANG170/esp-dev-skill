# 自动起网模式 (AUTO_START)

> **适用摘要**: 启用 `OPENTHREAD_BR_AUTO_START`，设备开机自动连 Wi-Fi、生成 Thread dataset、成为 Leader，并提供 Ethernet backbone 的唯一可行路径。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-thread-br/resources/`, source/examples in `repos/esp-thread-br/`, and this recipe path `repos/esp-thread-br/recipes/auto_start_mode.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图
- "开机自动连 Wi-Fi 并组网"
- "OPENTHREAD_BR_AUTO_START"
- "Ethernet backbone 自动启动"
- "SoftAP 首次配网"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考 | `examples/common/thread_border_router/src/border_router_launch.c`、`docs/en/dev-guide/build_and_run.rst` |

## 分步说明

### 1. 启用自动模式

```bash
idf.py menuconfig
# ESP Thread Border Router Example ->
#   [*] Enable the automatic start mode in Thread Border Router.   # OPENTHREAD_BR_AUTO_START
```

### 2. 配 Wi-Fi 凭据（两种来源）

**方式 A：menuconfig 预置**
```
# Example Connection Configuration -> WiFi SSID / WiFi Password
```
> 运行时优先用 NVS 中的 Wi-Fi 信息，没有才用 `EXAMPLE_WIFI_SSID/PASSWORD`。

**方式 B：SoftAP 首次配网（`OPENTHREAD_BR_SOFTAP_SETUP`）**
```bash
# ESP Thread Border Router Example ->
#   [*] Enable SoftAP Wi-Fi configuration mode    # OPENTHREAD_BR_SOFTAP_SETUP
```
首次启动 AP `ESP-ThreadBR-XXXX`，浏览器连 `http://192.168.4.1` 配网：

```c
esp_br_wifi_config_start();
esp_br_wifi_config_get_configured_wifi(ssid, sizeof(ssid),
                                       password, sizeof(password), 0);
esp_br_wifi_config_stop();
```

### 3. Ethernet backbone（必须 AUTO_START）

Ethernet backbone 不能手动起，必须 `OPENTHREAD_BR_AUTO_START=y`，否则 `app_main` 直接 `#error`。配置：
```
[*] connect using Ethernet interface    # EXAMPLE_CONNECT_ETHERNET
[ ] connect using WiFi interface        # EXAMPLE_CONNECT_WIFI 关闭
```
官方 BR 板 Sub-Ethernet (W5500) 引脚见 `sdkconfig.defaults`：SPI2 / SCLK=GPIO21 / MOSI=GPIO45 / MISO=GPIO38 / CS=GPIO41 / INT=GPIO39 / RST=GPIO40 / PHY addr=1。

### 4. 自动起网内部流程（`ot_br_init`）

```c
static void ot_br_init(void *ctx) {
#if CONFIG_EXAMPLE_CONNECT_WIFI
    // 1. 取/连 Wi-Fi（NVS 优先，否则 SoftAP 或 Kconfig）
    char wifi_ssid[32], wifi_password[64];
    if (esp_ot_wifi_config_get_ssid(wifi_ssid) == ESP_OK) {
        esp_ot_wifi_config_get_password(wifi_password);
        esp_ot_wifi_connect(wifi_ssid, wifi_password);
    } else {
#if CONFIG_OPENTHREAD_BR_SOFTAP_SETUP
        esp_br_wifi_config_start();
        esp_br_wifi_config_get_configured_wifi(wifi_ssid, ..., 0);
        esp_br_wifi_config_stop();
#else
        strncpy(wifi_ssid, CONFIG_EXAMPLE_WIFI_SSID, ...);
#endif
        wifi_config_save_and_connect(wifi_ssid, wifi_password);
    }
#elif CONFIG_EXAMPLE_CONNECT_ETHERNET
    ESP_ERROR_CHECK(example_ethernet_connect());
#endif

    // 2. 绑 backbone + 初始化 BR
    esp_openthread_lock_acquire(portMAX_DELAY);
    esp_openthread_set_backbone_netif(get_example_netif());
    ESP_ERROR_CHECK(esp_openthread_border_router_init());
#if CONFIG_EXAMPLE_CONNECT_WIFI
    esp_ot_wifi_border_router_init_flag_set(true);
#endif

    // 3. dataset：没有就随机生成（网络名 ESP-BR-<MAC>）
    otOperationalDatasetTlvs dataset;
    if (otDatasetGetActiveTlvs(esp_openthread_get_instance(), &dataset) != OT_ERROR_NONE) {
        otOperationalDataset new_dataset;
        otDatasetCreateNewNetwork(esp_openthread_get_instance(), &new_dataset);
        // ... 设置 mNetworkName = "ESP-BR-XXXX"
        otDatasetConvertToTlvs(&new_dataset, &dataset);
    }
    ESP_ERROR_CHECK(esp_openthread_auto_start(&dataset));
    esp_openthread_lock_release();
    vTaskDelete(NULL);
}
```

`launch_openthread_border_router` 在 `xTaskCreate(ot_br_init, "ot_br_init", 6144, NULL, 4, NULL)` 中调度。

### 5. 预置 Thread dataset（可选）

```
# Component config -> OpenThread -> Thread Operational Dataset
#   OPENTHREAD_NETWORK_NAME / OPENTHREAD_NETWORK_CHANNEL / OPENTHREAD_NETWORK_KEY ...
```
不预置时自动随机生成。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| Ethernet 模式编译报 `#error` | 关了 AUTO_START 又用 Ethernet | Ethernet 必须 `OPENTHREAD_BR_AUTO_START=y` |
| 自动模式连不上 Wi-Fi 后挂死 | Wi-Fi 配置错误未重启重试 | 仓库逻辑连不上会 `esp_restart()` 重试，确认 NVS/Kconfig 凭据正确 |
| 自动生成的网络名重复 | 默认用 MAC 后 4 hex | `ot_br_init` 用 `ESP-BR-%02X%02X`，碰撞概率极低；如需固定名走 `OPENTHREAD_NETWORK_NAME` |
| SoftAP 模式连不上 | 浏览器没连 AP `ESP-ThreadBR-XXXX` | 手机/电脑先连该 AP 再访问 192.168.4.1 |

## 参考
- `examples/common/thread_border_router/src/border_router_launch.c`
- `examples/basic_thread_border_router/main/esp_ot_br.c`
- `components/esp_ot_br_server/include/esp_br_wifi_config.h`
- `docs/en/dev-guide/build_and_run.rst`（2.1.3.1 / 2.1.3.2）
