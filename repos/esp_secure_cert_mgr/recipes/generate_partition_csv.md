# 用 configure_esp_secure_cert.py 生成 / 签名 / 解析分区

> **适用摘要**: 主机端使用 `tools/configure_esp_secure_cert.py` 工具，通过命令行参数或 CSV 配置文件生成 `esp_secure_cert.bin`，可选签名（Secure Boot V2）与解析已有镜像。这是出厂烧录与 QEMU 测试的标准流程。

## 触发意图

- "生成 esp_secure_cert 分区"
- "configure_esp_secure_cert.py 用法"
- "用 CSV 配置证书分区"
- "给分区签名"
- "解析 esp_secure_cert.bin"

## 前置条件

| 条件 | 要求 |
|---|---|
| 工具 | `pip install esp-secure-cert-tool`，或直接用 `tools/configure_esp_secure_cert.py` |
| 输入 | CA 证书、设备证书、私钥（PEM/DER），或一份 CSV |
| 参考 | `espressif-repos/esp_secure_cert_mgr/tools/README.md`、`docs/esp_secure_cert_tools/configure_esp_secure_cert_csv.md`、`configure_esp_secure_cert_parser.md` |

## 分步说明

### 1. 命令行生成（cust_flash_tlv，DS 外设）

```bash
# 先准备凭据（tools/README.md 给出的 openssl 步骤）
# 1) 生成 CA
openssl req -newkey rsa:2048 -nodes -keyout prvtkey.pem -x509 -days 3650 -out cacert.pem -subj "/CN=Test CA"
# 2) 生成客户端私钥
openssl genrsa -out client.key
# 3) 生成设备证书
openssl req -out client.csr -key client.key -new
openssl x509 -req -days 365 -in client.csr -CA cacert.pem -CAkey prvtkey.pem -sha256 -CAcreateserial -out client.crt

# 生成分区（默认 TLV，烧到 0xD000，含 DS 配置）
configure_esp_secure_cert.py -p /dev/ttyUSB0 \
    --keep_ds_data_on_host --efuse_key_id 1 \
    --ca-cert cacert.pem --device-cert client.crt --private-key client.key \
    --target_chip esp32c3 --secure_cert_type cust_flash_tlv --configure_ds
```

> 输出目录 `esp_secure_cert_data/` 含 `esp_secure_cert.bin`，并自动追加 `ESP_SECURE_CERT_INTEGRITY_TLV`。

### 2. CSV 方式生成（推荐，支持自定义数据 / 多种私钥类型）

CSV 字段顺序：`tlv_type,tlv_subtype,data_value,data_type,priv_key_type,algorithm,key_size,efuse_id,efuse_key`

**明文 RSA 私钥示例**（来自 `configure_esp_secure_cert_csv.md`）：
```csv
tlv_type,tlv_subtype,data_value,data_type,priv_key_type,algorithm,key_size,efuse_id,efuse_key
ESP_SECURE_CERT_CA_CERT_TLV,0,cacert.pem,file,,,,,
ESP_SECURE_CERT_DEV_CERT_TLV,0,client.crt,file,,,,,
ESP_SECURE_CERT_PRIV_KEY_TLV,0,client.key,file,plaintext,RSA,2048,,
```

**ECDSA 外设示例**：
```csv
ESP_SECURE_CERT_PRIV_KEY_TLV,0,ecdsa_client.key,file,ecdsa_peripheral,ECDSA,256,1,ecdsa_efuse.key
```

**DS 外设示例**：
```csv
ESP_SECURE_CERT_PRIV_KEY_TLV,0,client.key,file,rsa_ds,RSA,2048,1,
```

**自定义数据示例**：
```csv
ESP_SECURE_CERT_USER_DATA_1_TLV,0,"Device Model: ESP32-S3-DevKit-C",string,,,,
ESP_SECURE_CERT_USER_DATA_2_TLV,0,DEADBEEFCAFEBABE1234567890ABCDEF,hex,,,,
```

> `data_type` 支持 `file`/`string`/`hex`/`base64`；但 `priv_key`/`ca_cert`/`device_cert` 仅支持 `file`/`string`。运行示例 CSV 见 `examples/esp_secure_cert_app/esp_secure_cert_config_examples.csv`。

生成命令：
```bash
configure_esp_secure_cert.py --esp_secure_cert_csv config.csv \
    --target_chip esp32c3 --port /dev/ttyUSB0
```

### 3. 给已有镜像签名（Secure Boot V2）

```bash
configure_esp_secure_cert.py \
    --esp-secure-cert-file path/to/esp_secure_cert.bin \
    --secure-sign \
    --signing-key-file signing_private_key.pem \
    --signing-scheme rsa3072   # 或 ecdsa192 / ecdsa256 / ecdsa384
```

> 多个 `--signing-key-file` 生成多个签名块（subtype 0/1/2）。输出 `esp_secure_cert_signed_partition.bin` 与 `signature_block_*.bin`。

### 4. 解析已有镜像

```bash
configure_esp_secure_cert.py --parse_bin esp_secure_cert_data/esp_secure_cert.bin
```

输出 `esp_secure_cert_parsed_data/`：各 TLV 的 PEM/DER/bin 文件、`esp_secure_cert_parsed.csv`（可重建分区）、`tlv_entries_raw.txt`（偏移/类型/长度/CRC 摘要）。

### 5. 常用附加参数

| 参数 | 作用 |
|---|---|
| `--sec_cert_part_offset 0xD000` | 自定义烧录偏移 |
| `--skip_flash` | 只生成不烧录（适合 flash 加密开发模式） |
| `--target_chip esp32c3` | 目标芯片 |
| `--port /dev/ttyUSB0` 或 `socket://localhost:5555` | 串口 / QEMU |

### 6. QEMU 测试流程（来自 tools/README.md）

合并镜像 → 制作 `qemu_efuse.bin` → 以 download mode 启动 qemu（TCP 5555）→ 用 `--port socket://localhost:5555 --skip_flash` 生成 → `esptool write_flash 0xD000` → 切 boot mode 运行。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| CSV 列数不对 | 私钥行少填 `algorithm/key_size` | 严格按 9 列；非私钥行后 5 列留空 |
| `data_type` 不支持 | 私钥用了 `hex` | 私钥/证书仅 `file`/`string` |
| 生成失败 | eFuse key 已存在 / DS 配置冲突 | 换 `--efuse_key_id`，或对开发用 `--keep_ds_data_on_host` |
| 烧录后读不到 | 偏移与 `partitions.csv` 不一致 | `--sec_cert_part_offset` 与分区表 offset 对齐 |
| 签名校验不过 | signing scheme 与 secure boot 不匹配 | 选 `rsa3072`/`ecdsa256` 等与 bootloader 一致 |
| 解析缺文件 | 类型为自定义数据不生成单独文件 | 自定义数据只在 CSV 里体现 |

## 参考

- `espressif-repos/esp_secure_cert_mgr/tools/configure_esp_secure_cert.py`
- `espressif-repos/esp_secure_cert_mgr/tools/README.md`（安装、命令行、QEMU）
- `espressif-repos/esp_secure_cert_mgr/docs/esp_secure_cert_tools/configure_esp_secure_cert_csv.md`（CSV 字段）
- `espressif-repos/esp_secure_cert_mgr/docs/esp_secure_cert_tools/configure_esp_secure_cert_parser.md`（解析输出）
- `espressif-repos/esp_secure_cert_mgr/docs/esp_secure_cert_tools/secure_verification.md`（签名命令）
- `espressif-repos/esp_secure_cert_mgr/examples/esp_secure_cert_app/esp_secure_cert_config_examples.csv`（示例 CSV）
