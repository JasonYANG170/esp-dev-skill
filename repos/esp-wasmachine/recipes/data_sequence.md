# data_sequence：VM 与 WASM 应用的参数序列化

> **适用摘要**: 使用 `wasmachine_data_sequence` 组件（`data_seq`）在固件/VM 与 WASM 应用之间序列化传递参数，是 native 模块 `ioctl`/属性容器的底层机制。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-wasmachine/resources/`, source/examples in `repos/esp-wasmachine/`, and this recipe path `repos/esp-wasmachine/recipes/data_sequence.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "data_seq 怎么用"
- "ioctl 参数怎么传"
- "VM 与 WASM 传参"
- "wasmachine_data_sequence"

## 前置条件

| 条件 | 要求 |
|---|---|
| 组件 | `wasmachine_data_sequence`（`wasmachine_core` 已依赖，`override_path: ../wasmachine_data_sequence`） |
| 头文件 | `#include "data_seq.h"`（固件侧）/ `wm_ext_wasm_native_common.h`（native 模块侧） |

## 分步说明

### 1. 数据模型

来源 `components/wasmachine_data_sequence/include/data_seq.h`：

```c
typedef uint16_t data_seq_type_t;
typedef uint16_t data_seq_size_t;

typedef struct data_seq_frame {
    data_seq_type_t type;   /* 帧类型：每个 type 必须唯一 */
    data_seq_size_t size;   /* 帧大小 */
    uintptr_t       ptr;    /* 帧指针（push 时拷贝进去） */
} data_seq_frame_t;

typedef struct data_seq {
    uint32_t          version;   /* DATA_SEQ_V_1 = 0x1 */
    uint32_t          num;        /* 最大帧数（= alloc 时的 num） */
    uint32_t          index;      /* 当前空闲帧指针 */
    data_seq_frame_t  frame[0];
} data_seq_t;
```

### 2. 核心 API

```c
data_seq_t *data_seq_alloc(uint32_t num);          /* 创建：num = 最大可 push 帧数 */
void        data_seq_free(data_seq_t *ds);
void        data_seq_reset(data_seq_t *ds);

int data_seq_push(data_seq_t *ds, data_seq_type_t type,
                  data_seq_size_t size, const void *data);   /* 0 / -EINVAL / -ENOSPC */
int data_seq_pop(data_seq_t *ds, data_seq_type_t type,
                 data_seq_size_t size, void *data);          /* 0 / -EINVAL / -ENOENT */
int data_seq_update_frame_data(data_seq_t *ds, data_seq_type_t type,
                               data_seq_size_t size, void *data);
```

便捷宏（用 `sizeof` 自动算长度）：

```c
DATA_SEQ_PUSH(ds, t, v)
DATA_SEQ_POP(ds, t, v)
DATA_SEQ_UPDATE(ds, t, v)
DATA_SEQ_FORCE_PUSH(ds, t, v)   /* 失败 assert */
DATA_SEQ_FORCE_POP(ds, t, v)
DATA_SEQ_FORCE_UPDATE(ds, t, v)
```

### 3. 规则

- **每个 `type` 必须互不相同**（header 注释明确）。
- `push` 后的源数据在外部被释放前要确保已 `pop`（push 是拷贝）。
- `pop` 仅在 `type` 与 `size` 均匹配时成功，否则 `-ENOENT`。
- `alloc(num)`：若要序列化一个结构体，`num` 通常取结构体字段数。

### 4. 用法示例（固件侧）

```c
#include "data_seq.h"

enum { TYPE_FREQ = 1, TYPE_DUTY = 2 };

data_seq_t *ds = data_seq_alloc(2);
uint32_t freq = 1000, duty = 50;
DATA_SEQ_PUSH(ds, TYPE_FREQ, freq);
DATA_SEQ_PUSH(ds, TYPE_DUTY, duty);

/* 传给对端后 */
uint32_t out_freq = 0, out_duty = 0;
DATA_SEQ_POP(ds, TYPE_FREQ, out_freq);
DATA_SEQ_POP(ds, TYPE_DUTY, out_duty);

data_seq_free(ds);
```

### 5. 在 native 模块中的用法

native wrapper 收到 WASM 传来的 `char *va_args` 后，用帮助函数取出并做 WASM→native 地址映射（`components/wasmachine_ext_wasm_native/include/wm_ext_wasm_native_common.h`）：

```c
int   wm_ext_data_seq_addr_wasm2c(wasm_exec_env_t exec_env, data_seq_t *ds);
data_seq_t *wm_ext_wasm_native_get_data_seq(wasm_exec_env_t exec_env, char *va_args);
```

典型模式：

```c
static int my_dev_ioctl(wasm_exec_env_t exec_env, int fd, int cmd, char *va_args)
{
    data_seq_t *ds = wm_ext_wasm_native_get_data_seq(exec_env, va_args);
    if (!ds) return -1;

    uint32_t freq = 0;
    if (data_seq_pop(ds, TYPE_FREQ, sizeof(freq), &freq) != 0) return -1;
    /* 用 freq 配置硬件... */
    return 0;
}
```

> 这是 `wm_ext_wasm_gpio/i2c/spi/ledc_ioctl` 的统一传参机制（见 `recipes/ext_vfs.md`）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `pop` 返回 `-ENOENT` | type 不唯一或 size 不匹配 | 每个 type 唯一；push/pop 的 size 一致 |
| `push` 返回 `-ENOSPC` | 超过 `alloc(num)` 容量 | 增大 `num` |
| `FORCE_*` 触发 assert | 同上 | 仅在确信不会失败时用 FORCE 版本 |
| native 端读到乱数据 | 未做 WASM→native 地址映射 | 用 `wm_ext_wasm_native_get_data_seq` / `wm_ext_data_seq_addr_wasm2c` |

## 参考

- `components/wasmachine_data_sequence/include/data_seq.h` — 全部 API 与宏
- `components/wasmachine_data_sequence/src/data_seq.c` — 实现
- `components/wasmachine_data_sequence/test/test_data_seq.c` — 单元测试用法
- `components/wasmachine_ext_wasm_native/include/wm_ext_wasm_native_common.h` — native 帮助函数
- `recipes/ext_vfs.md` — ioctl 经 data_seq 传参
