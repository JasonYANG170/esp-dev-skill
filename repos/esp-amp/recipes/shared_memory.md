# 共享内存与 SysInfo

> **适用摘要**: 通过 SysInfo 在 maincore 分配共享内存块并分配 ID，subcore 按 ID 查询获取，实现跨核数据共享。这是所有 ESP-AMP IPC 组件的基础。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-amp/resources/`, source/examples in `repos/esp-amp/`, and this recipe path `repos/esp-amp/recipes/shared_memory.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "跨核共享数据"
- "maincore 传数据给 subcore"
- "SysInfo 分配"
- "共享内存"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `examples/rpmsg_send_recv/`、`examples/event/` |
| Kconfig | `CONFIG_ESP_AMP_HP_SHARED_MEM_SIZE` 足够大（内部组件已占用部分） |

## 分步说明

### 1. 定义共享数据结构与 SysInfo ID（common/sys_info.h）

```c
/* common/sys_info.h —— 双核共享 */
#pragma once

/* 用户 ID 范围：0x0000 ~ 0xfeff；0xff00~0xffff 为 ESP-AMP 内部保留 */
#define SYS_INFO_ID_PERSON_1   0x0001

typedef struct {
    char name[16];
    uint32_t age;
} Person_t;
```

### 2. maincore 分配并初始化（必须在 start_subcore 之前）

```c
#include "esp_amp.h"
#include "sys_info.h"

void app_main(void)
{
    assert(esp_amp_init() == 0);

    /* 在 HP RAM 分配（需要原子性的对象必须用 HP RAM） */
    Person_t *person = (Person_t *)esp_amp_sys_info_alloc(
        SYS_INFO_ID_PERSON_1, sizeof(Person_t), SYS_INFO_CAP_HP);
    assert(person != NULL);

    /* 写入内容（分配阶段无并发，可安全写入） */
    strncpy(person->name, "esp-amp", sizeof(person->name));
    person->age = 1;

    /* 之后启动 subcore ... */
}
```

### 3. subcore 按 ID 获取

```c
#include "esp_amp.h"
#include "sys_info.h"

int main(void)
{
    assert(esp_amp_init() == 0);

    Person_t *person = (Person_t *)esp_amp_sys_info_get(
        SYS_INFO_ID_PERSON_1, NULL, SYS_INFO_CAP_HP);
    if (person == NULL) {
        printf("SUB: failed to get person\r\n");
        return -1;
    }
    printf("SUB: name=%s, age=%d\r\n", person->name, (int)person->age);
    /* ... */
}
```

### 4. SysInfo 池选择

| 池 | 宏 | 适用 | 原子操作 |
|---|---|---|---|
| HP RAM 共享内存 | `SYS_INFO_CAP_HP` | 通用跨核通信、需要原子性的对象 | 支持 |
| RTC RAM 共享内存 | `SYS_INFO_CAP_RTC` | 仅 LP subcore；light sleep 期间需保留且不要求原子性的数据 | 不支持 |

### 5. 调试

```c
esp_amp_sys_info_dump();   // 打印所有 SysInfo 条目（ID/size/addr）
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| subcore `esp_amp_sys_info_get` 返回 NULL | maincore 未在 start_subcore 前分配该 ID | 确保 alloc 在 `esp_amp_start_subcore()` 之前完成 |
| SysInfo alloc 返回 NULL | 共享内存不足 | 增大 `CONFIG_ESP_AMP_HP_SHARED_MEM_SIZE`；内部已占用 6 个保留 ID |
| ID 冲突 | 使用了保留 ID（0xff00~0xffff） | 用户 ID 限定在 `0x0000`~`0xfeff` |
| 数据竞争/一致性破坏 | 把需要原子性的对象放 RTC RAM | virtqueue/event/sw_intr 对象必须用 `SYS_INFO_CAP_HP` |
| SysInfo 无法释放 | 设计如此 | SysInfo 一旦分配保留到系统复位，无 free API |

## 参考

- `espressif-repos/esp-amp/docs/shared_memory.md`
- `espressif-repos/esp-amp/components/esp_amp/include/esp_amp_sys_info.h`
- `examples/rpmsg_send_recv/common/event.h`（跨核共享头文件示例）
