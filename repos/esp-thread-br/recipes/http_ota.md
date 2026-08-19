# HTTPS OTA 升级 Border Router

> **适用摘要**: 用本地 openssl HTTPS 服务器下发 `ota_with_rcp_image`，触发 BR 自身 OTA（必要时连带 RCP）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-thread-br/resources/`, source/examples in `repos/esp-thread-br/`, and this recipe path `repos/esp-thread-br/recipes/http_ota.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图
- "OTA 升级 Border Router"
- "ota download"
- "本��� HTTPS OTA 服务器"
- "ota_with_rcp_image"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `CONFIG_OPENTHREAD_CLI_OTA=y`（`ota` 命令）、`CONFIG_CREATE_OTA_IMAGE_WITH_RCP_FW=y` |
| 分区 | 双 OTA 分区 (`ota_0`/`ota_1`) + `otadata` |
| 证书 | 自签证书替换 `server_certs/ca_cert.pem` |
| 参考 | `docs/en/dev-guide/ota_update.rst`、`docs/en/dev-guide/ota_local_server.rst` |

## 分步说明

### 1. 启用相关选项

```bash
# sdkconfig.defaults 默认已含：
CONFIG_OPENTHREAD_CLI_OTA=y
CONFIG_OPENTHREAD_RCP_COMMAND=y
# 构建期生成 OTA bundle：
CONFIG_CREATE_OTA_IMAGE_WITH_RCP_FW=y
```

### 2. 在 app_main 注入信任证书

```c
extern const uint8_t server_cert_pem_start[] asm("_binary_ca_cert_pem_start");
extern const uint8_t server_cert_pem_end[]   asm("_binary_ca_cert_pem_end");

// app_main:
#if CONFIG_OPENTHREAD_CLI_OTA
    esp_set_ota_server_cert((char *)server_cert_pem_start);   // esp_ot_ota_commands.h
#endif
```

### 3. 生成自签证书并起 HTTPS 服务器

```bash
mkdir ota_server_storage && cd ota_server_storage
openssl req -x509 -newkey rsa:2048 -keyout ca_key.pem -out ca_cert.pem -days 365 -nodes
# Common Name (CN) 必须与运行服务器的主机名一致
openssl s_server -WWW -key ca_key.pem -cert ca_cert.pem -port 8070
```

> CN 必须与 `ota download` 的 URL 主机名一致，否则证书校验失败。

### 4. 替换 BR 工程证书并重建

```bash
cp /path/to/ota_server_storage/ca_cert.pem \
   esp-thread-br/examples/basic_thread_border_router/server_certs/ca_cert.pem
idf.py fullclean
idf.py -p PORT flash monitor
```

### 5. 复制 OTA 镜像到服务器目录

```bash
cp esp-thread-br/examples/basic_thread_border_router/build/ota_with_rcp_image ota_server_storage/
```

### 6. 在 BR 触发下载

```
> ot ota download https://${HOST_URL}:8070/ota_with_rcp_image
```
下载完成后 BR 自动重启刷入新分区；若 RCP 版本变化则一并更新 RCP。

### 7. OTA 镜像结构（来自 create_ota_image.py）

| filetag | 内容 |
|---|---|
| 0 | RCP 版本 |
| 1 | RCP flash arguments |
| 2 | RCP bootloader |
| 3 | RCP partition table |
| 4 | RCP firmware |
| 5 | Border Router firmware |

镜像头：`0xff | header size | 0`，后接各子文件 `(filetag, size, offset)`。

### 8. 高层 OTA 辅助 API（程序内触发）

`components/esp_br_http_ota/include/esp_br_http_ota.h`：
```c
esp_err_t esp_br_http_ota(esp_http_client_config_t *http_config);
#define OTA_MAX_WRITE_SIZE 16
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| TLS 握手失败 | 证书 CN 不匹配 / 未替换 | CN 与 URL 主机名一致；替换 `server_certs/ca_cert.pem` 后 `fullclean` |
| `ota` 命令不存在 | 未启用 | `CONFIG_OPENTHREAD_CLI_OTA=y` |
| 找不到 `ota_with_rcp_image` | 构建期未打包 | `CONFIG_CREATE_OTA_IMAGE_WITH_RCP_FW=y` |
| 下载后不重启 | 分区表缺双 OTA | `partitions.csv` 含 `ota_0`/`ota_1`/`otadata` |
| 镜像过大 | 4MB Flash 不够 | 用 8MB 板；4MB 变体把 ota 分区改 1500K |

## 参考
- `docs/en/dev-guide/ota_update.rst`（2.3）
- `docs/en/dev-guide/ota_local_server.rst`（2.2）
- `components/esp_br_http_ota/include/esp_br_http_ota.h`
- `components/esp_ot_cli_extension/include/esp_ot_ota_commands.h`
- `examples/basic_thread_border_router/server_certs/`
- `examples/basic_thread_border_router/README.md`（Updating the border router from HTTPS server）
