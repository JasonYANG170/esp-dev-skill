# 工厂分区与认证凭据（esp-matter-mfg-tool）

> **适用摘要**: 用 `esp-matter-mfg-tool` 生成含 VID/PID/CD/DAC/passcode/discriminator 的工厂分区二进制，烧录到 fctry 分区，并在 menuconfig 选择对应的 Factory/Secure Cert Provider，让设备用真实（或测试）凭据入网。

## 触发意图

- "生成工厂分区"
- "esp-matter-mfg-tool"
- "DAC / CD / PAI 怎么烧"
- "生产凭据"
- "VID/PID 配置"
- "CONFIG_ENABLE_ESP32_FACTORY_DATA_PROVIDER"

## 前置条件

| 条件 | 要求 |
|---|---|
| 工具 | `esp-matter-mfg-tool`（esp-matter-tools 的 Python 包） |
| 工具 | `chip-cert`（生成测试 CD），位于 `connectedhomeip/connectedhomeip/out/host` |
| 分区表 | `partitions.csv` 含 `fctry` 分区（见 `examples/light/partitions.csv`，默认偏移 `0x3E0000`） |

## 分步说明

### 1. 生成测试 Certification Declaration（CD）

如果没有正式 CD，用 `chip-cert gen-cd` 生成测试 CD（VID/PID 与设备一致）。来自 `docs/en/developing.rst`：

```bash
cd connectedhomeip/connectedhomeip
gn gen out/host && ninja -C out/host   # 若尚未构建 host 工具

out/host/chip-cert gen-cd -f 1 -V 0xFFF1 -p 0x8001 -d 0x0016 \
    -c "CSA00000SWC00000-01" -l 0 -i 0 -n 1 -t 0 \
    -K credentials/test/certification-declaration/Chip-Test-CD-Signing-Key.pem \
    -C credentials/test/certification-declaration/Chip-Test-CD-Signing-Cert.pem \
    -O TEST_CD_FFF1_8001.der
```

`-V`（Vendor ID）、`-p`（Product ID）必须与后续 DAC、Basic cluster 上报的值一致。

### 2. 用 esp-matter-mfg-tool 生成工厂分区

```bash
export PATH=$PATH:$PWD/connectedhomeip/connectedhomeip/out/host

esp-matter-mfg-tool --passcode 89674523 \
    --discriminator 2245 \
    -cd TEST_CD_FFF1_8001.der \
    -v 0xFFF1 --vendor-name Espressif \
    -p 0x8001 --product-name Bulb \
    --hw-ver 1 --hw-ver-str DevKit
```

输出（注意 QR / manual code，入网时用）：

```text
[INFO] Generated QR code: MT:-24J06PF150QJ850Y10
[INFO] Generated manual code: 20489154736
[INFO] Generated output files at: out/fff1_8001/<uuid>
```

`<uuid>.bin` 即工厂分区二进制。

### 3. （可选）用 PAI 签名 DAC

若要带 PAI 签名的 DAC（更接近生产），加 `-k`（PAI 私钥）与 `-c`（PAI 证书）、`--pai`：

```bash
esp-matter-mfg-tool -cn "My bulb" -v 0xFFF2 -p 0x8001 --pai \
    -k path/to/Chip-Test-PAI-FFF2-8001-Key.pem \
    -c path/to/Chip-Test-PAI-FFF2-8001-Cert.pem \
    -cd path/to/Chip-Test-CD-FFF2-8001.der
```

### 4. 烧录工厂分区

先确认 `partitions.csv` 里 `fctry` 分区的偏移（`examples/light/partitions.csv` 为 `0x3E0000`）：

```bash
esptool.py -p (PORT) write_flash 0x3E0000 path/to/<uuid>.bin
```

若用 `esp_secure_cert` 分区（Pre-Provisioned 模组），按其偏移（`0xd000`）烧 `secure_cert_partition.bin`：

```bash
esptool.py -p (PORT) write_flash 0xd000 secure_cert_partition.bin
```

### 5. 在 menuconfig 选 Provider

`Component config → ESP Matter` 下有四组 `choice`（来自 `components/esp_matter/Kconfig`）：

| Provider 类别 | 工厂分区选项 | 安全证书选项 | 说明 |
|---|---|---|---|
| DAC | `CONFIG_FACTORY_PARTITION_DAC_PROVIDER` | `CONFIG_SEC_CERT_DAC_PROVIDER` | Attestation 凭据 |
| Commissionable Data | `CONFIG_FACTORY_COMMISSIONABLE_DATA_PROVIDER` | `CONFIG_SEC_CERT_COMMISSIONABLE_DATA_PROVIDER` | passcode/discriminator/salt/verifier |
| Device Instance Info | `CONFIG_FACTORY_DEVICE_INSTANCE_INFO_PROVIDER` | `CONFIG_SEC_CERT_DEVICE_INSTANCE_INFO_PROVIDER` | VID/PID/序列号等 |
| Device Info | `CONFIG_FACTORY_DEVICE_INFO_PROVIDER` | — | fixed/user labels |

启用工厂分区 Provider 的总开关：

```
CONFIG_ENABLE_ESP32_FACTORY_DATA_PROVIDER=y
CONFIG_ENABLE_ESP32_DEVICE_INSTANCE_INFO_PROVIDER=y   # 如需 Device Instance Info
CONFIG_ENABLE_TEST_SETUP_PARAMS=n                      # 关掉测试凭据
```

使用 `esp_secure_cert` 分区时：

```
CONFIG_ESP_SECURE_CERT_DS_PERIPHERAL=n                 # 不用 DS 外设（H2/P4/C5 有 ECDSA 时可 =y）
CONFIG_SEC_CERT_DAC_PROVIDER=y
```

### 6. 用工厂分区的凭据入网

用 mfg_tool 输出的 QR / manual code 入网：

```text
chip-tool interactive start
pairing code-wifi 0x7283 <ssid> <passphrase> MT:-24J06PF150QJ850Y10
```

自定义 PAA（DAC 由自定义 PAI 签发）时加 `--paa-trust-store-path`：

```bash
chip-tool pairing ble-wifi 1234 <ssid> <passphrase> <passcode> <discriminator> \
    --paa-trust-store-path /path/to/PAA-Certificates/
```

### 7. （可选）把 CD 嵌入固件而非工厂分区

启用 `CONFIG_ENABLE_SET_CERT_DECLARATION_API` 后，可在代码里用 `SetCertificationDeclaration()` 设置 CD（见 `examples/light/main/app_main.cpp`）：

```cpp
#ifdef CONFIG_ENABLE_SET_CERT_DECLARATION_API
extern const uint8_t cd_start[] asm("_binary_certification_declaration_der_start");
extern const uint8_t cd_end[]   asm("_binary_certification_declaration_der_end");
const chip::ByteSpan cdSpan(cd_start, cd_end - cd_start);

auto *dac_provider = get_dac_provider();
#ifdef CONFIG_SEC_CERT_DAC_PROVIDER
static_cast<ESP32SecureCertDACProvider *>(dac_provider)->SetCertificationDeclaration(cdSpan);
#elif defined(CONFIG_FACTORY_PARTITION_DAC_PROVIDER)
static_cast<ESP32FactoryDataProvider *>(dac_provider)->SetCertificationDeclaration(cdSpan);
#endif
#endif
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 入网 `SRC_ERR_INVALID_SETUP_CSR` | 测试凭据已关但工厂分区没烧 | 二选一：保留 `CONFIG_ENABLE_TEST_SETUP_PARAMS=y` 或烧 fctry 分区 |
| VID/PID 不匹配 | CD/DAC/Basic cluster 三处 VID/PID 必须一致 | gen-cd、mfg-tool、`basic_information` 用同一组 |
| fctry 偏移错 | partitions.csv 与烧录地址不一致 | 核对 `CONFIG_CHIP_FACTORY_NAMESPACE_PARTITION_LABEL` 与分区偏移 |
| Secure Cert DS 崩溃 | H2/P4/C5 有 ECDSA 外设但配置不符 | `CONFIG_ESP_SECURE_CERT_DS_PERIPHERAL` 按芯片能力设 |
| Custom Provider 不生效 | 在 `esp_matter::start()` 之后才设 | `set_custom_*_provider()` 必须在 start 之前 |

## 参考

- `D:/esp-skill/espressif-repos/esp-matter/docs/en/developing.rst` — Factory Data Providers / Using esp_secure_cert partition / Certification Declaration / Factory Partition
- `D:/esp-skill/espressif-repos/esp-matter/examples/light/partitions.csv`（fctry 分区偏移 `0x3E0000`）
- `D:/esp-skill/espressif-repos/esp-matter/examples/light/main/app_main.cpp`（`SetCertificationDeclaration` 用法）
- `D:/esp-skill/espressif-repos/esp-matter/components/esp_matter/Kconfig`（四组 Provider `choice`）
- `D:/esp-skill/espressif-repos/esp-matter/docs/en/production.rst`（生产考量）
