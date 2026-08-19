# MSC 大容量存储（U 盘）

> **适用摘要**: 用 TinyUSB MSC 类实现 USB U 盘，包括 RAM/Flash 后端、必需的 SCSI 回调（read10/write10/capacity/inquiry）、多 LUN 与可写控制。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/tinyusb/resources/`, source/examples in `repos/tinyusb/`, and this recipe path `repos/tinyusb/recipes/msc_device.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "USB U 盘"
- "TinyUSB MSC"
- "tud_msc_read10_cb"
- "USB 大容量存储设备"
- "RAM disk / Flash disk"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/device/cdc_msc/src/msc_disk.c`（RAM 盘）、`examples/device/msc_dual_lun/` |
| 配置项 | `CFG_TUD_MSC=1`、`CFG_TUD_MSC_EP_BUFSIZE`（建议 512） |

## 分步说明

### 1. tusb_config.h 使能 MSC

```c
#define CFG_TUD_MSC               1
#define CFG_TUD_MSC_EP_BUFSIZE    512
```

### 2. RAM 盘后端定义

```c
#define DISK_BLOCK_NUM   16    // 8KB（Windows 能挂载的最小尺寸）
#define DISK_BLOCK_SIZE  512

// 预填充 FAT12 引导扇区（见 examples/device/cdc_msc/src/msc_disk.c）
static uint8_t msc_disk[DISK_BLOCK_NUM][DISK_BLOCK_SIZE] = { 0 };
```

### 3. 必需 SCSI 回调

```c
// 容量查询（块数 / 块大小）
void tud_msc_capacity_cb(uint8_t lun, uint32_t *block_count, uint16_t *block_size) {
  (void)lun;
  *block_count = DISK_BLOCK_NUM;
  *block_size  = DISK_BLOCK_SIZE;
}

// 是否就绪（介质在位）
bool tud_msc_test_unit_ready_cb(uint8_t lun) {
  (void)lun;
  return true;   // false 时用 tud_msc_set_sense 标记 NOT_READY
}

// 读
int32_t tud_msc_read10_cb(uint8_t lun, uint32_t lba, uint32_t offset,
                          void *buffer, uint32_t bufsize) {
  (void)lun;
  if (lba >= DISK_BLOCK_NUM) {
    tud_msc_set_sense(lun, SCSI_SENSE_ILLEGAL_REQUEST, 0x3a, 0x00);
    return -1;
  }
  memcpy(buffer, &msc_disk[lba][offset], bufsize);
  return (int32_t)bufsize;
}

// 写
int32_t tud_msc_write10_cb(uint8_t lun, uint32_t lba, uint32_t offset,
                           uint8_t *buffer, uint32_t bufsize) {
  (void)lun;
  if (lba >= DISK_BLOCK_NUM) {
    tud_msc_set_sense(lun, SCSI_SENSE_ILLEGAL_REQUEST, 0x3a, 0x00);
    return -1;
  }
  memcpy(&msc_disk[lba][offset], buffer, bufsize);
  return (int32_t)bufsize;
}

// 是否可写（只读盘返回 false）
bool tud_msc_is_writable_cb(uint8_t lun) {
  (void)lun;
  return true;
}
```

### 4. INQUIRY / 其他 SCSI 回调

```c
// 经典写法（旧版接口）
void tud_msc_inquiry_cb(uint8_t lun, uint8_t vendor_id[8],
                        uint8_t product_id[16], uint8_t product_rev[4]) {
  (void)lun;
  memcpy(vendor_id,   "TinyUSB", 8);
  memcpy(product_id,  "MassStorage", 16);
  memcpy(product_rev, "1.0", 4);
}

// 未识别的 SCSI 命令
int32_t tud_msc_scsi_cb(uint8_t lun, uint8_t const scsi_cmd[16],
                        void *buffer, uint16_t bufsize) {
  (void)bufsize; (void)buffer;
  tud_msc_set_sense(lun, SCSI_SENSE_ILLEGAL_REQUEST, 0x20, 0x00);
  return -1;
}

// 可选：弹出/装入、起停
bool tud_msc_start_stop_cb(uint8_t lun, uint8_t power_condition,
                           bool start, bool load_eject) { (void)lun; (void)power_condition; (void)start; (void)load_eject; return true; }

// 读/写完成通知（可选）
void tud_msc_read10_complete_cb(uint8_t lun)  { (void)lun; }
void tud_msc_write10_complete_cb(uint8_t lun) { (void)lun; }
```

### 5. 多 LUN

```c
#define CFG_TUD_MSC   2   // 2 个逻辑单元

uint8_t tud_msc_get_maxlun_cb(void) {
  return CFG_TUD_MSC;     // 返回 LUN 数
}

// 各回调按 lun 区分后端
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| Windows 提示“请插入磁盘” | 容量小于 8KB 或 `test_unit_ready` 返回 false | `DISK_BLOCK_NUM*SIZE ≥ 8KB`，就绪返回 true |
| 写入后重启丢失 | 后端是 RAM | 改用 Flash 后端，并在 `write10_cb` 里擦写 Flash |
| 设备管理器报错代码 10 | `read10/write10` 越界或返回值错 | 加边界检查，失败用 `tud_msc_set_sense` 并返回 -1 |
| 主机读不到容量 | `tud_msc_capacity_cb` 没设出参 | 必须写 `*block_count` / `*block_size` |
| 只读盘仍被写 | `tud_msc_is_writable_cb` 返回 true | 只读盘返回 false |

## 参考

- `examples/device/cdc_msc/src/msc_disk.c` — 完整 RAM 盘（含 FAT12 引导扇区）
- `examples/device/msc_dual_lun/` — 双 LUN
- `src/class/msc/msc_device.h` — MSC 设备 API 与回调声明
