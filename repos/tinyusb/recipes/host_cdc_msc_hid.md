# USB 主机（CDC/MSC/HID）

> **适用摘要**: 用 TinyUSB 主机栈枚举并通信外接 USB 设备，包括 `tuh_task` 主循环、挂载/卸载回调、CDC-ACM/HID/MSC host API 与异步传输回调。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/tinyusb/resources/`, source/examples in `repos/tinyusb/`, and this recipe path `repos/tinyusb/recipes/host_cdc_msc_hid.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "TinyUSB 主机"
- "tuh_task"
- "读 U 盘 / 读键盘（作主机）"
- "tuh_msc_read10 / tuh_hid_receive_report"
- "USB host CDC"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/host/cdc_msc_hid/`、`examples/host/msc_file_explorer/` |
| 配置项 | `CFG_TUH_ENABLED=1`、`CFG_TUH_CDC/MSC/HID/HUB` |

## 分步说明

### 1. tusb_config.h 使能主机栈

```c
#define CFG_TUH_ENABLED           1
#define CFG_TUH_HUB               1   // 支持 Hub（多级）
#define CFG_TUH_CDC               1
#define CFG_TUH_MSC               1
#define CFG_TUH_HID               1
// ESP32（无原生主机）用 MAX3421E via SPI：CFG_TUH_MAX3421=1
```

### 2. 主机主循环

```c
int main(void) {
  board_init();

  tusb_rhport_init_t host_init = {
    .role  = TUSB_ROLE_HOST,
    .speed = TUSB_SPEED_AUTO
  };
  tusb_init(BOARD_TUH_RHPORT, &host_init);
  board_init_after_tusb();

  while (1) {
    tuh_task();          // 主机任务（必须频繁调用）
    // cdc_app_task(); hid_app_task();
  }
}

// 设备挂载/卸载（设备地址分配后触发）
void tuh_mount_cb(uint8_t dev_addr) {
  (void)dev_addr;
}
void tuh_umount_cb(uint8_t dev_addr) {
  (void)dev_addr;
}
```

### 3. HID 主机：接收 report

```c
// 设备 HID 接口挂载（report descriptor 已解析）
void tuh_hid_mount_cb(uint8_t dev_addr, uint8_t idx,
                      uint8_t const *report_desc, uint16_t desc_len) {
  (void)report_desc; (void)desc_len;
  // 启动接收
  if (!tuh_hid_receive_report(dev_addr, idx)) {
    // 失败处理
  }
}

void tuh_hid_umount_cb(uint8_t dev_addr, uint8_t idx) {
  (void)dev_addr; (void)idx;
}

// 收到 report
void tuh_hid_report_received_cb(uint8_t dev_addr, uint8_t idx,
                                uint8_t const *report, uint16_t len) {
  // 处理 report ... 然后继续接收
  tuh_hid_receive_report(dev_addr, idx);
}
```

### 4. CDC 主机：读写串口设备

```c
void tuh_cdc_mount_cb(uint8_t idx) {
  // 设备 CDC 接口就绪
}
void tuh_cdc_umount_cb(uint8_t idx) {
  (void)idx;
}

// 写：tuh_cdc_write(idx, buf, len); tuh_cdc_write_flush(idx);
// 读：tuh_cdc_read_available(idx); tuh_cdc_read(idx, buf, bufsize);
// 设置波特率（异步，完成回调）
cdc_line_coding_t lc = { .bit_rate = 115200, .stop_bits = 0, .parity = 0, .data_bits = 8 };
tuh_cdc_set_line_coding(idx, &lc, NULL, 0);
```

### 5. MSC 主机：异步读写

```c
void tuh_msc_mount_cb(uint8_t dev_addr) {
  uint8_t const lun = 0;
  // 测试就绪 → 读容量 → 读写
  tuh_msc_test_unit_ready(dev_addr, lun, my_complete_cb, 0);
}
void tuh_msc_umount_cb(uint8_t dev_addr) { (void)dev_addr; }

// 异步读
tuh_msc_read10(daddr, lun, buffer, lba, block_count, my_complete_cb, 0);
// 异步写
tuh_msc_write10(daddr, lun, buffer, lba, block_count, my_complete_cb, 0);

// 完成回调签名
void my_complete_cb(uint8_t dev_addr, tuh_msc_complete_data_t const *cb_data) {
  // cb_data->cbw / cb_data->csw / cb_data->lun / cb_data->msc_res
}
```

### 6. 查询容量

```c
uint32_t blocks = tuh_msc_get_block_count(daddr, lun);
uint32_t bsize  = tuh_msc_get_block_size(daddr, lun);
uint8_t  maxlun = tuh_msc_get_maxlun(daddr);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 设备插上无反应 | `tuh_task()` 没在主循环 | while(1) 里加 `tuh_task()` |
| Hub 下设备不枚举 | 未开 Hub 支持 | `CFG_TUH_HUB=1` |
| HID 只收一帧 | 没在回调里续收 | `tuh_hid_report_received_cb` 末尾再调 `tuh_hid_receive_report` |
| MSC 读不到容量 | 没先 `test_unit_ready`/`read_capacity` | 按挂载→就绪→容量→读写顺序 |
| ESP32 主机不工作 | ESP32 无原生主机控制器 | 用 MAX3421E（`CFG_TUH_MAX3421=1`，via SPI） |
| 异步传输卡死 | 完成回调里阻塞 | 回调只做轻量处理，重活放主循环 |

## 参考

- `examples/host/cdc_msc_hid/src/main.c` — 主机主循环与挂载回调
- `examples/host/cdc_msc_hid/src/hid_app.c`、`cdc_app.c`、`msc_app.c` — 各类应用任务
- `examples/host/msc_file_explorer/` — U 盘文件浏览（FatFs）
- `src/class/cdc/cdc_host.h`、`src/class/msc/msc_host.h`、`src/class/hid/hid_host.h` — 主机类 API
