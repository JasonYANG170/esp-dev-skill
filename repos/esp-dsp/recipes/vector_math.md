# 向量数学与点积

> **适用摘要**: 用 `dsps_add_f32`/`dsps_sub_f32`/`dsps_mul_f32`/`dsps_mulc_f32`/`dsps_sqrt_f32` 做逐元素向量运算（含 step 参数），用 `dsps_dotprod_f32` 做点积。涵盖 f32 / s16 / s8 数据类型。

## 触发意图

- "向量加 / 减 / 乘"
- "点积 / dot product"
- "逐元素运算"
- "乘常数"
- "平方根"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/basic_math/main/dsps_math_main.c`、`examples/dotprod/main/dsps_dotproduct_main.c` |
| 头文件 | `esp_dsp.h`（含 `dsps_math.h`、`dsps_dotprod.h`） |

## 分步说明

### 逐元素运算（带 step）

`out[i*step_out] = input1[i*step1] <op> input2[i*step2]; i=[0..len)`。step 默认 1，设为 2 可直接操作交错复数。

```c
#include "esp_log.h"
#include "esp_dsp.h"

static const char *TAG = "main";
#define N 1024

__attribute__((aligned(16))) static float a[N], b[N], out[N];

void app_main(void)
{
    /* 加：out[i] = a[i] + b[i] */
    dsps_add_f32(a, b, out, N, 1, 1, 1);
    /* 减 */
    dsps_sub_f32(a, b, out, N, 1, 1, 1);
    /* 逐元素乘 */
    dsps_mul_f32(a, b, out, N, 1, 1, 1);
    /* 乘常数：out[i] = a[i] * C */
    dsps_mulc_f32(a, out, N, 3.14f, 1, 1);
}
```

### 直接操作交错复数（step=2）

`basic_math` 示例的核心技巧：把实部按 step 2 写入复数 buffer：

```c
__attribute__((aligned(16))) static float x1[N], wind[N], y_cf[N * 2];

/* y_cf 的偶数下标（实部）= x1 * wind；虚部下标先填 0 */
dsps_mul_f32(x1, wind, y_cf, N, 1, 1, 2);          /* step_out=2：写 y_cf[0],y_cf[2],... */
dsps_mulc_f32(&y_cf[1], &y_cf[1], N, 0, 2, 2);     /* 虚部清零：从 y_cf[1] 起 step 2 乘 0 */
```

### 点积

`dsps_dotprod_f32(src1, src2, &dest, len)` —— `dest += sum(src1[i]*src2[i])`。

```c
__attribute__((aligned(16))) static float in1[256], in2[256];
for (int i = 0; i < 256; i++) { in1[i] = 1; in2[i] = i; }

float result = 0;
unsigned int s = dsp_get_cpu_cycle_count();
dsps_dotprod_f32(in1, in2, &result, 101);   /* 计算 0+1+...+100 */
unsigned int e = dsp_get_cpu_cycle_count();
ESP_LOGI(TAG, "sum(0..100) = %f, %u cycles", result, e - s);
```

带 step 的点积：`dsps_dotprode_f32(src1, src2, &dest, len, step1, step2)`。

### 平方根（近似）

```c
float in[N], out[N];
dsps_sqrt_f32(in, out, N);          /* 逐元素近似 sqrt */
float r = dsps_sqrtf_f32_ansi(2.0f);          /* 标量近似 sqrt */
float inv = dsps_inverted_sqrtf_f32_ansi(2.0f); /* 1/sqrt(x) */
```

### 定点（s16 / s8）

定点逐元素运算多一个 `shift` 参数（结果右移位数）：

```c
int16_t a16[L], b16[L], out16[L];
dsps_add_s16(a16, b16, out16, L, 1, 1, 1, /*shift*/0);
dsps_sub_s16(a16, b16, out16, L, 1, 1, 1, 0);
int16_t C16 = 4; int16_t out2[L];
dsps_mulc_s16(a16, out2, L, C16, 1, 1);

int8_t a8[L], b8[L], out8[L];
dsps_add_s8(a8, b8, out8, L, 1, 1, 1, 0);
dsps_sub_s8(a8, b8, out8, L, 1, 1, 1, 0);
```

定点点积带 shift：

```c
int16_t dest16;
dsps_dotprod_s16(s1, s2, &dest16, len, /*shift*/15);

int32_t dest32;
dsps_dp_s8(s1_8, s2_8, &dest32, len);   /* 8 位点积，结果 32 位无移位 */
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 复数 buffer 写错位 | step_out 没设 2 | 交错复数用 `step_out=2`，虚部从 `&buf[1]` 起 |
| 定点结果溢出/为 0 | shift 不对 | s16 结果右移 `shift` 位；按动态范围调整 |
| `dsps_sqrt_f32` 精度差 | 它是近似实现 | 是近似（非 `sqrtf`），高精度需求用 `sinf`/`sqrtf` |
| 点积 `dest` 未初始化 | 当成输出却传随机值 | 点积是 `*dest += ...`（累加），调用前按需清零 |
| s8 点积返回 32 位无移位 | 误当 16 位用 | `dsps_dp_s8` 写入 `int32_t*`，不右移 |

## 参考

- `examples/basic_math/main/dsps_math_main.c` — `dsps_mul_f32`/`dsps_mulc_f32` 操作交错复数
- `examples/dotprod/main/dsps_dotproduct_main.c` — `dsps_dotprod_f32` 周期测量
- `modules/math/*/include/dsps_*.h` — add/sub/mul/mulc/addc/sqrt 各数据类型
- `modules/dotprod/include/dsps_dotprod.h` — `dsps_dotprod_f32`、`dsps_dotprode_f32`、`dsps_dotprod_s16`、`dsps_dp_s8`
