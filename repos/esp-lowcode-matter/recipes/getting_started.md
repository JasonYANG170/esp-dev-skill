# 本地终端环境搭建与首次烧录

> **适用摘要**: 在本地终端用 ESP-IDF v5.3 + ESP-AMP + esp-lowcode-matter 搭建开发环境，完成 Prepare Device、Upload Configuration、Upload Code 全流程，把一个示例产品跑起来。

## 触发意图

- "lowcode 怎么开始"
- "本地搭建 esp-lowcode-matter 环境"
- "如何烧录 lowcode 产品"
- "Upload Configuration / Upload Code"
- "first time setup lowcode"

## 前置条件

| 条件 | 要求 |
|---|---|
| 硬件 | ESP32-C6 开发板（如 ESP32-C6-MINI-1），USB 连接电脑 |
| 主机 | 已安装 git、python3、esptool.py；USB 串口权限已配置 |
| 参考文档 | 仓库 `docs/getting_started_terminal.md`、`docs/hardware_setup.md` |

## 分步说明

### 1. 克隆并安装依赖

```sh
# ESP-IDF v5.3
git clone -b release/v5.3 https://github.com/espressif/esp-idf.git --recursive
cd esp-idf && export ESP_IDF_PATH=$(pwd) && ./install.sh && . ./export.sh && cd ..

# ESP-AMP
git clone -b main https://github.com/espressif/esp-amp.git --recursive
cd esp-amp && export ESP_AMP_PATH=$(pwd) && git submodule update --init --recursive && cd ..

# esp-lowcode-matter
git clone -b main https://github.com/espressif/esp-lowcode-matter.git --recursive
cd esp-lowcode-matter && git submodule update --init --recursive && ./install.sh && . ./export.sh && cd ..
```

> 注意：IDF 与 ESP-AMP 会被 LowCode 的 `install.sh`/`export.sh` 间接使用，但**不**加入工作区。

### 2. 选择产品

```sh
export SELECTED_PRODUCT=light_cw_pwm   # 或 socket / temperature_sensor / template 等
cd $LOW_CODE_PATH/products/$SELECTED_PRODUCT
```

可选产品（来自 `products/README.md`）：`light_cw_pwm`、`light_rgbcw_ws2812`、`socket`、`socket_2_channel`、`temperature_sensor`、`temperature_sensor_with_display`、`occupancy_sensor`、`thermostat`、`template`。

### 3. Prepare Device（每块板仅一次）—— 烧 HP 预编译镜像

```sh
cd $LOW_CODE_PATH/pre_built_binaries
esptool.py erase_flash
esptool.py write_flash $(cat flash_args)
```

### 4. Upload Configuration（每块板仅一次）—— 证书 + QR + data_model.bin

先取 MAC（格式 `0123456789ABCDEF`，去冒号大写）：

```sh
cd $LOW_CODE_PATH/tools/mfg
esptool.py chip_id        # 输出形如 MAC: 01:23:45:67:89:ab:cd:ef
export MAC_ADDRESS=0123456789ABCDEF
```

生成并烧录：

```sh
./mfg_low_code.sh $LOW_CODE_PATH/products/$SELECTED_PRODUCT esp32c6 $MAC_ADDRESS
esptool.py write_flash 0xD000  $LOW_CODE_PATH/products/$SELECTED_PRODUCT/configuration/output/$MAC_ADDRESS/${MAC_ADDRESS}_esp_secure_cert.bin \
                       0x1F2000 $LOW_CODE_PATH/products/$SELECTED_PRODUCT/configuration/output/$MAC_ADDRESS/${MAC_ADDRESS}_fctry.bin
```

QR 码生成于 `.../configuration/output/$MAC_ADDRESS/qr_code.png`，配网时用生态 App（Apple/Google/Amazon/Home Assistant/Samsung）扫描。

### 5. Upload Code —— 构建 + 烧 LP 固件

```sh
cd $LOW_CODE_PATH/products/$SELECTED_PRODUCT
idf.py set-target esp32c6
idf.py build
esptool.py write_flash 0x20C000 build/$SELECTED_PRODUCT.bin
```

### 6. Console 看日志

```sh
python3 -m esp_idf_monitor
```

成功后会看到 `app_main: Starting low code` 与后续事件日志。配网/控制见 `docs/device_setup.md`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `system_setup` 后立即 panic | Prepare Device 未烧 HP 镜像 | 先在 `pre_built_binaries/` 执行 `write_flash $(cat flash_args)` |
| `esptool.py` 连不上 | 串口权限/驱动 | Linux 加 udev 规则、Windows 装 USB 驱动；查 `docs/hardware_setup.md` |
| 配网 QR 扫不出 | 没跑 Upload Configuration | 跑 `mfg_low_code.sh` 并烧 `esp_secure_cert.bin` + `fctry.bin` |
| 改了 zap 不生效 | data_model.bin 未更新 | 重跑 Upload Configuration |
| `idf.py build` 找不到组件 | 未 `. ./export.sh` 或组件未在 REQUIRES | 先 source export.sh；补 `main/CMakeLists.txt` 的 REQUIRES |

## 参考

- `docs/getting_started_terminal.md`
- `docs/hardware_setup.md`
- `products/<your_product>/README.md`
