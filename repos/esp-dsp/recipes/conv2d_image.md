# 2D 卷积 / 图像处理

> **适用摘要**: 用 `dspi_conv_f32` 对 2D 图像（`image2d_t` 结构）做卷积，复现 Matlab `conv2(A,B,'same')`；通过 `stride_x`/`stride_y`/`step_x`/`step_y` 字段表达行宽与子采样，适用于相机/传感器阵列的边缘检测、模糊、锐化等核卷积。

## 触发意图

- "2D 卷积"
- "图像卷积"
- "dspi_conv"
- "image2d_t"
- "卷积核 / Sobel / 边缘检测"
- "conv2 / 'same' 模式"
- "传感器阵列 / 像素阵列滤波"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/conv2d/main/conv2d_main.c` |
| 头文件 | `esp_dsp.h`（含 `dspi_conv.h`、`dsp_types.h` 中的 `image2d_t`） |
| 对齐 | `data` 指针用 `memalign(16, ...)` 分配 |

## 分步说明

### `image2d_t` 字段语义（最常被搞错）

来自 `modules/common/include/dsp_types.h`：

```c
typedef struct image2d_s {
    void *data;    // 指向像素数据（float / int16 / int8 均可）
    int step_x;    // X 方向相邻元素的跨度（字面元素步长，通常为 1）
    int step_y;    // Y 方向相邻元素的跨度（通常为 1）
    int stride_x;  // 行宽：一行里元素个数 * step_x + padding（可理解为 row pitch）
    int stride_y;  // 列方向重复系数（通常为 1；Point[x,y] 实际寻址见下）
    int size_x;    // 图像宽度（参与运算的有效列数）
    int size_y;    // 图像高度（参与运算的有效行数）
} image2d_t;
/* 寻址公式：Point[x,y] = data[stride_y*y*step_y + stride_x*x*step_x]
   注：头文件注释写作 width*y*step_y + x*step_x，实现中等价于 stride_x 充当 width */
```

要点：`stride_x` **不是** "子采样步长"，而是**行宽（row pitch）**；真正的子采样由 `step_x`/`step_y` 控制（一般都用 1）。`size_x`/`size_y` 是参与运算的 ROI 大小，可以小于 `stride_x`（用于只卷图像的一个窗口）。

### `same` 模式输出尺寸（函数会覆写 size 字段）

直接取自 `modules/conv/float/dspi_conv_f32_ansi.c` 的前两行：

```c
esp_err_t dspi_conv_f32_ansi(const image2d_t *in_image, const image2d_t *filter, image2d_t *out_image)
{
    out_image->size_x = in_image->size_x;   // 输出尺寸 = 输入尺寸
    out_image->size_y = in_image->size_y;
    ...
}
```

即 `dspi_conv_f32` **固定为 `'same'` 模式**：输出图像大小 = 输入图像大小，滤波器在边缘做局部覆盖（边界处只用核内有效的那些像素参与累加，核的中心锚点为 `rest = (filter_size - 1) >> 1`）。**没有 `'full'` 模式**；想要 `conv2(A,B,'full')`（输出 `sizeA + sizeB - 1`）请自行扩边后再调本函数。

输出缓冲的 `data` 数组至少要能容纳 `stride_x * size_y`（即 `out.stride_x * out.size_y`，本例为 `10*10`）个 float；构造时 `size_x`/`size_y` 写 0 没关系（函数会按输入覆写），但 `stride_x`/`step_x`/`step_y`/`stride_y` 必须正确。

### 完整示例（复现 `conv2(ones(8), ones(4), 'same')`）

直接取自 `examples/conv2d/main/conv2d_main.c`，输出与 `conv2d/README.md` 列出的结果一致（中心 16、角点 4）：

```c
#include <stdio.h>
#include <malloc.h>
#include "esp_dsp.h"
#include "dsp_tests.h"

static const char *TAG = "main";

void app_main(void)
{
    ESP_LOGI(TAG, "Start Example.");
    int max_N = 100;

    /* 三个缓冲，16 字节对齐 */
    float *data1 = (float *)memalign(16, max_N * sizeof(float));
    float *data2 = (float *)memalign(16, max_N * sizeof(float));
    float *data3 = (float *)memalign(16, max_N * sizeof(float));

    /* 构造三个 image2d_t：{data, step_x, step_y, stride_x, stride_y, size_x, size_y}
       - image1: 8x8 输入
       - image2: 4x4 核
       - image3: 输出，stride_x=10 表示底层数组一行能放 10 个元素（>= 8 即可），
                 size_x/size_y 留 0，函数会按输入覆写成 8x8 */
    image2d_t image1 = {data1, 1, 1, 8, 8, 8, 8};    // 8x8
    image2d_t image2 = {data2, 1, 1, 4, 4, 4, 4};    // 4x4
    image2d_t image3 = {data3, 1, 1, 10, 10, 0, 0};  // 输出（size 由函数填）

    for (int i = 0; i < max_N; i++) { data1[i] = 0; data2[i] = 0; data3[i] = 0; }

    /* 把 image1 全填 1、image2 全填 1（复现 ones(8)、ones(4)） */
    for (int y = 0; y < image1.stride_y / image1.step_y; y++) {
        for (int x = 0; x < image1.stride_x / image1.step_x; x++) {
            data1[y * image1.stride_x * image1.step_y + x * image1.step_x] = 1;
        }
    }
    for (int y = 0; y < image2.stride_y / image2.step_y; y++) {
        for (int x = 0; x < image2.stride_x / image2.step_x; x++) {
            data2[y * image2.stride_x * image2.step_y + x * image2.step_x] = 1;
        }
    }

    /* 2D 卷积 */
    dspi_conv_f32(&image1, &image2, &image3);

    ESP_LOGI(TAG, "2D Convolution result.");
    /* 按寻址公式读取输出（注意 size_x/size_y 现在已被函数置为 8/8） */
    for (int y = 0; y < image3.size_y; y++) {
        printf("[%2i .. %2i, %2i]:  ", 0, image3.size_x, y);
        for (int x = 0; x < image3.size_x; x++) {
            printf("%2.0f, ", data3[y * image3.stride_x * image3.step_y + x * image3.step_x]);
        }
        printf("\n");
    }
    free(data1); free(data2); free(data3);
}
```

预期输出（与 README 一致）：角点 `(0,0)=9`、`(7,7)=4`、中心 `(3..4,3..4)=16`。

### Sobel / 边缘检测核（只改 data2 的内容）

`image2d_t` 不规定核的语义，把 `data2` 换成 Sobel-X 系数即可做边缘检测，核大小仍是 `size_x=3, size_y=3`：

```c
image2d_t sobel_x = {data2, 1, 1, 3, 3, 3, 3};
/* 核内容按 stride_x=3, step_x=1, step_y=1 排布：
   -1  0  1
   -2  0  2
   -1  0  1   */
float sobel_x_data[9] = {-1, 0, 1, -2, 0, 2, -1, 0, 1};
memcpy(data2, sobel_x_data, sizeof(sobel_x_data));
dspi_conv_f32(&image1, &sobel_x, &image3);
```

### 子采样（ROI / 抽行抽列）

利用 `step_x > 1` 可在 X 方向抽列；用更大的 `stride_x` 配合更小的 `size_x` 可只卷一个 ROI 窗口。例：对 16x16 图像每 2 列取 1 列、卷 8x8 的核：

```c
image2d_t sub = {data1, 2, 1, 16, 1, 8, 8};  /* step_x=2 表示隔列采样；size_x=8 取前 8 列 */
```

注意：`step` 影响寻址，写错会读到越界内存，不会报错（函数不检查边界）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 输出全是 0 | 输出 `image3` 的 `stride_x`/`step_x` 写成 0 | `stride_x` 至少要 `>= size_x`（输出底层数组行宽），`step_x`/`step_y` 用 1 |
| 输出尺寸对不上 | 以为函数给 `'full'`（`sizeA+sizeB-1`） | 本函数固定 `'same'`（输出 = 输入尺寸）；想要 `'full'` 自己扩边后调用 |
| 边缘值奇怪 | 误以为函数补零（zero-pad） | 函数对边缘只累加核内有效像素，等价于 Matlab `'same'` 不补零；需补零请自行扩边 |
| 越界 / 崩溃 | `stride_x` 小于实际行宽，或 `data3` 数组太小 | `data3` 至少 `stride_x * size_y` 个元素；示例用 10x10 底数组存 8x8 结果 |
| `step_x`/`step_y` 写错 | 当成 "字节步长" 或省略 | 这两个是元素步长，常规图像都用 1；只有抽行/抽列时才 > 1 |
| `size_x`/`size_y` 没初始化就被读 | 在调用前读输出 image 的 size | 函数会覆写输出 `size_x/size_y`，调用前可填 0，**调用后**再读 |

## 参考项目

- `examples/conv2d/main/conv2d_main.c` — `ones(8) ⊛ ones(4)` 完整示例，输出与 Matlab `conv2(...,'same')` 一致
- `examples/conv2d/README.md` — 预期输出（中心 16、角点 4、边 12、角内 9/6）
- `modules/conv/include/dspi_conv.h` — `dspi_conv_f32` 声明（无论是否 `CONFIG_DSP_OPTIMIZED` 都映射到 `_ansi`，目前无 ae32/aes3/arp4 优化版本）
- `modules/conv/float/dspi_conv_f32_ansi.c` — 实现（含边缘局部覆盖逻辑与 size 覆写）
- `modules/common/include/dsp_types.h` — `image2d_t` 结构定义与寻址公式
- `modules/conv/test/test_dspi_conv_f32_ansi.c` — 单元测试，断言了角点/中心点的期望值
