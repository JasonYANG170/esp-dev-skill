# deflate 压缩烧录（节省传输时间）

> **适用摘要**: 把目标固件预先 zlib 压缩，用 `esp_loader_flash_deflate_start/write/finish` 烧录压缩流，目标端 stub 解压写入，显著减少 UART 传输量。需 stub 连接，且 deflate 路径不内部做 MD5，需单独 verify。

## 触发意图

- "压缩烧录"
- "deflate 烧录"
- "减少烧录传输量"
- "zlib 固件烧录"

## 前置条件

| 条件 | 要求 |
|---|---|
| 连接方式 | `esp_loader_connect_with_stub()`（serial 接口）；或 SDIO（自动 stub） |
| 数据 | 预压缩的 zlib 流 + 原始未压缩镜像 + 原始 MD5 |
| 参考示例 | `examples/esp32_deflate_example/` |

## 分步说明

### 1. 准备压缩镜像与原始 MD5

```c
extern const uint8_t app_bin[];             // 原始未压缩
extern const uint32_t app_bin_size;
extern const uint8_t app_deflated_bin[];    // 预压缩 zlib 流
extern const uint32_t app_deflated_bin_size;
extern const uint8_t app_bin_md5[];         // 原始镜像的 16 字节 MD5
#define DEFLATE_BLOCK_SIZE 1024
```

压缩脚本见 `examples/esp32_deflate_example/target-firmware/compress_firmware.py`。

### 2. 用 stub 连接

```c
if (connect_to_target_with_stub(&loader, 230400) != ESP_LOADER_SUCCESS) {
    return;
}
```

### 3. start（填 image_size=原始大小，compressed_size=压缩大小）

```c
esp_loader_flash_deflate_cfg_t cfg = {
    .offset          = 0x10000,
    .image_size      = app_bin_size,         // 未压缩大小
    .compressed_size = app_deflated_bin_size,// 压缩大小
    .block_size      = DEFLATE_BLOCK_SIZE,
};
esp_loader_error_t err = esp_loader_flash_deflate_start(&loader, &cfg);
if (err == ESP_LOADER_ERROR_UNSUPPORTED_FUNC) {
    printf("非 serial 接口或未用 stub，不支持 deflate\n");
    return err;
}
```

### 4. 循环写压缩块

```c
static uint8_t payload[DEFLATE_BLOCK_SIZE];
uint32_t off = 0;
while (off < cfg.compressed_size) {
    uint32_t n = MIN(cfg.compressed_size - off, DEFLATE_BLOCK_SIZE);
    memcpy(payload, app_deflated_bin + off, n);
    err = esp_loader_flash_deflate_write(&loader, &cfg, payload, n);
    if (err != ESP_LOADER_SUCCESS) {
        esp_loader_flash_deflate_finish(&loader, &cfg);
        return err;
    }
    off += n;
}
```

### 5. finish（不做 MD5，需单独 verify 明文）

```c
err = esp_loader_flash_deflate_finish(&loader, &cfg);
if (err != ESP_LOADER_SUCCESS) return err;

// deflate 不内部累积 MD5，用明文 MD5 校验落盘内容
err = esp_loader_flash_verify_known_md5(&loader, cfg.offset, cfg.image_size, app_bin_md5);
if (err != ESP_LOADER_SUCCESS) {
    printf("deflate 烧录后 MD5 校验失败\n");
    return err;
}
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `UNSUPPORTED_FUNC`（start） | 未用 stub / 非 serial 接口 | 先 `connect_with_stub`；SPI 不支持 |
| 校验失败 | 压缩流与原始镜像不同源 / MD5 错 | 用 bin2array 同源生成；重新压缩 |
| 解压中途错 | compressed_size / image_size 填反 | image_size=原始，compressed_size=压缩 |
| 体积没省 | 数据已高度压缩 | deflate 对已压缩二进制收益有限 |

## 参考

- `examples/esp32_deflate_example/main/main.c` — deflate 烧录 + verify 完整示例
- `examples/esp32_deflate_example/target-firmware/compress_firmware.py` — 预压缩脚本
- `include/esp_loader.h` — `esp_loader_flash_deflate_cfg_t`、deflate API 注释（明确不做内部 MD5）
- `recipes/connect_with_stub.md` — stub 连接前置
