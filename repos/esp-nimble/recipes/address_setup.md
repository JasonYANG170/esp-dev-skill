# 设备地址配置

> **适用摘要**: 为 NimBLE 设备配置本机 BLE 地址（public / 静态随机 / NRPA / RPA 隐私），并通过 `ble_hs_id_infer_auto` 推断 own_addr_type。

## 触发意图

- "设置 BLE 地址"
- "random address / 随机地址"
- "隐私 / RPA / NRPA"
- "BLE_OWN_ADDR"
- "ble_hs_id_infer_auto"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `apps/scanner/src/main.c`、`apps/advertiser/src/main.c` |
| 时序 | 地址相关 API 在 `on_sync` 中调用 |

## 分步说明

### 地址类型与 own_addr_type 常量

| own_addr_type 常量 | 含义 |
|---|---|
| `BLE_OWN_ADDR_PUBLIC` | 公共地址 |
| `BLE_OWN_ADDR_RANDOM` | 静态随机地址 |
| `BLE_OWN_ADDR_RPA_PUBLIC_DEFAULT` | RPA 隐私，身份地址为 public |
| `BLE_OWN_ADDR_RPA_RANDOM_DEFAULT` | RPA 隐私，身份地址为 random |

### 方式 A：自动推断（最常用，无需隐私）

适用于大多数外设 / 中心应用。Controller 提供公共地址时优先用 public。

```c
static uint8_t own_addr_type;

static void on_sync(void)
{
    int rc = ble_hs_util_ensure_addr(0);   /* 确保已配置身份地址 */
    assert(rc == 0);

    /* privacy=0：不启用 RPA */
    rc = ble_hs_id_infer_auto(0, &own_addr_type);
    assert(rc == 0);

    /* 之后把 own_addr_type 传给 ble_gap_adv_start / ble_gap_disc / ble_gap_connect */
    bleprph_advertise();
}
```

> 参考 `apps/bleprph/src/main.c` 的 `bleprph_on_sync`、`apps/peripheral/src/main.c` 的 `on_sync`。

### 方式 B：生成并设置随机地址（NRPA）

NRPA（Non-Resolvable Private Address）适用于一次性广播 / 扫描场景。设置后必须用 `BLE_OWN_ADDR_RANDOM` 作为 own_addr_type。

```c
static void set_ble_addr(void)
{
    ble_addr_t addr;
    int rc;

    /* nrpa=1：生成不可解析私有地址 */
    rc = ble_hs_id_gen_rnd(1, &addr);
    assert(rc == 0);

    rc = ble_hs_id_set_rnd(addr.val);
    assert(rc == 0);
}

static void on_sync(void)
{
    set_ble_addr();
    /* 因为是 NRPA，own_addr_type 必须为 RANDOM */
    ble_gap_disc(BLE_OWN_ADDR_RANDOM, 1000, &scan_params, scan_event, NULL);
}
```

> 参考 `apps/scanner/src/main.c`、`apps/advertiser/src/main.c`。

### 方式 C：启用 RPA 隐私

```c
static void on_sync(void)
{
    int rc = ble_hs_util_ensure_addr(0);   /* 确保 IRK / 身份地址就绪 */
    assert(rc == 0);

    /* privacy=1：启用 RPA，infer_auto 返回 RPA_*_DEFAULT */
    rc = ble_hs_id_infer_auto(1, &own_addr_type);
    assert(rc == 0);

    advertise();
}
```

### 关键 API

| 函数 | 说明 |
|---|---|
| `ble_hs_id_gen_rnd(int nrpa, ble_addr_t *out_addr)` | 生成随机地址（nrpa≠0 为 NRPA） |
| `ble_hs_id_set_rnd(const uint8_t *rnd_addr)` | 设置控制器随机地址 |
| `ble_hs_id_infer_auto(int privacy, uint8_t *out_addr_type)` | 推断 own_addr_type |
| `ble_hs_id_copy_addr(uint8_t id_addr_type, uint8_t *out_id_addr, int *out_is_nrpa)` | 拷贝当前身份地址 |
| `ble_hs_util_ensure_addr(int prefer_random)` | 确保至少存在一种身份地址 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `ble_hs_id_set_rnd` 返回非 0 | 地址不符合随机地址规范 | 用 `ble_hs_id_gen_rnd` 生成，勿手填 |
| 广播用 NRPA 但 own_addr_type 写成 PUBLIC | 类型不匹配 | NRPA / 静态随机地址都用 `BLE_OWN_ADDR_RANDOM` |
| `ble_hs_id_infer_auto` 返回 `BLE_HS_ENOADDR` | 未先 ensure_addr 且无 public | 先 `ble_hs_util_ensure_addr(0)` |
| 开启 RPA 后对端无法识别 | 未分发 IRK / 未绑定 | 完成 SMP 配对并交换 IRK |

## 参考

- `apps/scanner/src/main.c` — NRPA 生成与扫描
- `apps/advertiser/src/main.c` — NRPA 广播
- `docs/ble_setup/ble_addr.rst` — 地址配置官方说明
- 头文件：`nimble/host/include/host/ble_hs_id.h`
