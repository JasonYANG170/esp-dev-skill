# 矩阵运算（C 接口与 C++ dspm::Mat）

> **适用摘要**: 用 C 接口 `dspm_mult_f32` / `dspm_mult_ex_f32` / `dspm_add/sub/mulc` 做矩阵运算，或用 C++ `dspm::Mat` 类（运算符重载、`solve`/`roots`/`inverse`/`det`/`t`/`eye`/`ones`）做线性代数求解。

## 触发意图

- "矩阵乘法"
- "解线性方程组 A*x=b"
- "dspm::Mat"
- "矩阵求逆 / 行列式"
- "3x3 / 4x4 专用乘法"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/matrix/main/dspm_matrix_main.cpp` |
| C++ Mat | 源文件须为 `.cpp`，`app_main` 用 `extern "C"` |
| 头文��� | `esp_dsp.h`（C++ 下自动 include `mat.h`） |

## 分步说明

### C 接口：通用矩阵乘

`dspm_mult_f32(A, B, C, m, n, k)` 计算 `C[m][k] = A[m][n] * B[n][k]`，行优先。

```c
#include "esp_dsp.h"

/* C[2][3] = A[2][2] * B[2][3] */
float A[2*2] = {1,2, 3,4};
float B[2*3] = {1,0,1, 0,1,1};
float C[2*3];
dspm_mult_f32(A, B, C, /*m*/2, /*n*/2, /*k*/3);
```

带 padding/stride 的子矩阵乘：

```c
/* A_padd/B_padd/C_padd 为各行末尾的 padding 元素数 */
dspm_mult_ex_f32(A, B, C, m, n, k, A_padd, B_padd, C_padd);
```

### C 接口：定点与定长优化

```c
int16_t As[3*3], Bs[3*3], Cs[3*3];
dspm_mult_s16(As, Bs, Cs, 3, 3, 3, /*shift*/0);

/* 定长优化版（无 shift 参数） */
float A3[9], B3[3], C3[3];
dspm_mult_3x3x1_f32(A3, B3, C3);    /* A[3x3] * B[3x1] = C[3x1] */
dspm_mult_3x3x3_f32(A3, B3, C3);    /* 3x3 * 3x3 */
float A4[16], B4[4], C4[4];
dspm_mult_4x4x1_f32(A4, B4, C4);
dspm_mult_4x4x4_f32(A4, B4, C4);

/* int8 矩阵乘向量：C[1][M] = A[M][N] * B[1][N]，结果 int32 */
int8_t Ai[M*N], Bi[N]; int32_t Ci[M];
dspm_mult_mxn_1xm_int8(Ai, Bi, Ci, M, N);
```

### C++ dspm::Mat：基本运算与求解

取自官方 `matrix` 示例：

```cpp
#include "esp_log.h"
#include "esp_dsp.h"

static const char *TAG = "main";
extern "C" void app_main();

void app_main()
{
    int M = 3, N = 3;
    dspm::Mat A(M, N);
    dspm::Mat x(N, 1);
    for (int m = 0; m < M; m++) {
        for (int n = 0; n < N; n++) A(m, n) = N * m + n;
        x(m, 0) = m;
    }
    A(0, 0) = 10; A(0, 1) = 11;

    dspm::Mat b = A * x;                  /* 运算符重载（内部走优化 dspm_mult） */
    dspm::Mat x1_ = dspm::Mat::solve(A, b);  /* 高斯消元求解 A*x=b */
    dspm::Mat x2_ = dspm::Mat::roots(A, b);  /* 另一种解法 */

    std::cout << x << x1_ << x2_;
}
```

### dspm::Mat 常用方法

| 方法 / 运算符 | 说明 |
|---|---|
| `Mat(rows, cols)` / `Mat(data, rows, cols)` | 构造（内部分配 / 外部 buffer） |
| `M(r,c)` / `M(i,j) = v` | 元素访问 |
| `A + B`, `A - B`, `A * B`, `A*C`, `C*A` | 矩阵与常数运算 |
| `A += B`, `A -= B`, `A *= B`, `A *= C`, `A /= C`, `A ^= n` | 复合赋值 / 幂 |
| `A.t()` | 转置 |
| `Mat::eye(n)`, `Mat::ones(n)`, `Mat::ones(r,c)` | 单位阵 / 全 1 阵 |
| `Mat::solve(A,b)`, `Mat::roots(A,y)`, `Mat::bandSolve(A,b,k)` | 解方程组 |
| `A.inverse()`, `A.pinv()`, `A.det(n)` | 逆 / 伪逆 / 行列式 |
| `A.normalize()`, `A.norm()` | 归一化 / 范数 |
| `A.swapRows(r1,r2)`, `A.clear()` | 行交换 / 清零 |
| `Mat::dotProduct(A,B)`, `Mat::augment(A,B)` | 点积 / 增广矩阵 |
| `M.getROI(...)`, `M.Get(...)`, `M.block(...)` | 子矩阵 / ROI |
| `std::cout << M` | 打印矩阵 |

### 用 Mat 做状态估计示例（旋转矩阵 / 四元数）

`ekf` 类提供的静态方法可配合 Mat 使用（详见 kalman recipe）：

```cpp
dspm::Mat Rm = dspm::Mat::eye(3);
dspm::Mat q  = ekf::rotm2quat(Rm);          /* 旋转矩阵 → 四元数 */
dspm::Mat eu = ekf::quat2eul(q.data);       /* 四元数 → 欧拉角 */
float xyz[3] = {0.1f, 0.2f, 0.3f};
dspm::Mat Re = ekf::eul2rotm(xyz);          /* 欧拉角 → 旋转矩阵 */
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| C 文件里用 `dspm::Mat` | mat.h 仅 C++ | 源文件改为 `.cpp`，`app_main` 加 `extern "C"` |
| 维度不匹配 | `m/n/k` 传错 | `dspm_mult_f32(A,B,C,m,n,k)` 中 `A[m][n]`、`B[n][k]`、`C[m][k]` |
| 子矩阵越界 | `dspm_mult_ex_f32` padding 误用 | padding 是每行末尾额外元素数，非总大小 |
| `Mat::solve` 返回 NaN | A 奇异 / 接近奇异 | 检查 A 是否满秩；示例中特意改 `A(0,0)=10` 避免奇异 |
| int8 矩阵结果类型错 | 当成 16 位 | `dspm_mult_mxn_1xm_int8` 结果为 `int32_t` |

## 参考

- `examples/matrix/main/dspm_matrix_main.cpp` — `dspm::Mat` 构造、`A*x`、`solve`/`roots`
- `modules/matrix/include/dspm_matrix.h` — 聚合 add/addc/mult/mulc/sub
- `modules/matrix/mul/include/dspm_mult.h` — `dspm_mult_f32`/`_ex_f32`/`_s16`/`_3x3x1`/`_3x3x3`/`_4x4x1`/`_4x4x4`/`_mnx_1xm_int8`
- `modules/matrix/include/mat.h` — `dspm::Mat` 完整类定义
