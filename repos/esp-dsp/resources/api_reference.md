# ESP-DSP API Quick Reference

> 全部签名摘自 `modules/*/include/*.h`。函数名一律给出扩展名无关的形式（`dsps_*` / `dspm_*` / `dspi_*`），由头文件 `#define` 按目标与 `CONFIG_DSP_OPTIMIZED` 映射到 `_ansi`/`_ae32`/`_aes3`/`_arp4`。所有 C 接口返回 `esp_err_t`（成功为 `ESP_OK`），FIR/resampler 的 exec 类函数除外（返回样本数）。

## 顶层头文件

```c
#include "esp_dsp.h"   /* 聚合：common, dotprod, math, fir, resampler, biquad,
                          biquad_gen, wind, conv, corr, d_gen, h_gen, tone_gen,
                          snr, sfdr, fft2r, fft4r, dct, matrix, view, dspi_* */
```

C++ 下额外可用：`#include "mat.h"`（由 esp_dsp.h 自动 include）、`#include "ekf.h"` / `#include "ekf_imu13states.h"`。

## Common / 错误码 (`dsp_common.h`, `dsp_err_codes.h`)

```c
bool dsp_is_power_of_two(int x);     /* 是否 2 的幂 */
int  dsp_power_of_two(int x);        /* 求幂次 */
unsigned int dsp_get_cpu_cycle_count; /* 宏：按 IDF 版本映射到 esp_cpu_get_cycle_count 等 */

/* 错误码 */
ESP_OK
ESP_ERR_DSP_BASE              0x70000
ESP_ERR_DSP_INVALID_LENGTH    (BASE+1)
ESP_ERR_DSP_INVALID_PARAM     (BASE+2)
ESP_ERR_DSP_PARAM_OUTOFRANGE  (BASE+3)
ESP_ERR_DSP_UNINITIALIZED     (BASE+4)
ESP_ERR_DSP_REINITIALIZED     (BASE+5)
ESP_ERR_DSP_ARRAY_NOT_ALIGNED (BASE+6)
```

## 数据类型 (`dsp_types.h`)

```c
typedef union sc16_u { struct { int16_t re; int16_t im; }; uint32_t data; } sc16_t;
typedef union fc32_u { struct { float re; float im; };       uint64_t data; } fc32_t;

typedef struct image2d_s {
    void *data; int step_x; int step_y; int stride_x; int stride_y; int size_x; int size_y;
} image2d_t;   /* Point[x,y] = data[stride_y*y*step_y + stride_x*x*step_x] */
```

## FFT Radix-2 (`dsps_fft2r.h`)

```c
/* float complex (fc32) */
esp_err_t dsps_fft2r_init_fc32(float *fft_table_buff, int table_size);  /* NULL + CONFIG_DSP_MAX_FFT_SIZE = 内部分配 */
void     dsps_fft2r_deinit_fc32(void);
esp_err_t dsps_fft2r_fc32(float *data, int N);            /* data: Re,Im 交错, 长度 2*N */
esp_err_t dsps_bit_rev_fc32(float *data, int N);
esp_err_t dsps_bit_rev2r_fc32(float *data, int N);
esp_err_t dsps_cplx2reC_fc32(float *data, int N);         /* 一路复数 → 两路实信号 */
esp_err_t dsps_gen_w_r2_fc32(float *w, int N);
extern float *dsps_fft_w_table_fc32; extern int dsps_fft_w_table_size;

/* int16 complex (sc16) */
esp_err_t dsps_fft2r_init_sc16(int16_t *fft_table_buff, int table_size);
void     dsps_fft2r_deinit_sc16(void);
esp_err_t dsps_fft2r_sc16(int16_t *data, int N);
esp_err_t dsps_bit_rev_sc16(int16_t *data, int N);
esp_err_t dsps_cplx2reC_sc16(int16_t *data, int N);
esp_err_t dsps_gen_w_r2_sc16(int16_t *w, int N);
extern int16_t *dsps_fft_w_table_sc16;
```

> sc16（定点 Q15 复数 FFT）与 fc32 是**两套独立的表**，各自 init/deinit，互不影响。实时音频流（I2S 麦克风 → 加窗 → FFT → 拆两路实信号 → dB → 滑动平均）的完整链路见 `recipes/audio_spectrum_streaming.md`，参考应用 `applications/spectrum_box_lite/main/main.c`。

## FFT Radix-4 (`dsps_fft4r.h`)

```c
esp_err_t dsps_fft4r_init_fc32(float *fft_table_buff, int max_fft_size); /* 表大小 = 4*max_fft_size */
void     dsps_fft4r_deinit_fc32(void);
esp_err_t dsps_fft4r_fc32(float *data, int N);
esp_err_t dsps_bit_rev4r_fc32(float *data, int N);
esp_err_t dsps_cplx2real_fc32(float *data, int N);        /* 复数 FFT 结果 → 实频谱（实输入场景） */
extern float *dsps_fft4r_w_table_fc32; extern int dsps_fft4r_w_table_size;
```

## DCT / DST (`dsps_dct.h`)

```c
esp_err_t dsps_dct_f32(float *data, int N);       /* DCT-II, data 大小 2*N, 基于 FFT */
esp_err_t dsps_dct_inv_f32(float *data, int N);   /* 逆 DCT (type III), data 大小 2*N */
esp_err_t dsps_dctiv_f32(float *data, int N);     /* DCT-IV, data 大小 N */
esp_err_t dsps_dstiv_f32(float *data, int N);     /* DST-IV, data 大小 N */
/* 参考实现（非优化） */
esp_err_t dsps_dct_f32_ref(float *data, int N, float *result);
esp_err_t dsps_dct_inverce_f32_ref(float *data, int N, float *result);  /* 注意原拼写 */
```

## FIR 滤波器 (`dsps_fir.h`)

```c
typedef struct fir_f32_s {
    float *coeffs; float *delay; int N; int pos; int decim; int16_t use_delay;
    int delay_size; int interp; int interp_pos; int start_pos;
} fir_f32_t;

typedef struct fir_s16_s {
    int16_t *coeffs; int16_t *delay; int16_t coeffs_len, pos, decim, d_pos, shift;
    int32_t *rounding_buff; int32_t rounding_val; int16_t free_status;
    int16_t delay_size, interp, interp_pos, start_pos;
} fir_s16_t;

/* 标准 FIR */
esp_err_t dsps_fir_init_f32(fir_f32_t *fir, float *coeffs, float *delay, int coeffs_len);
esp_err_t dsps_fir_f32(fir_f32_t *fir, const float *input, float *output, int len);
esp_err_t dsps_fir_f32_free(fir_f32_t *fir);

/* 抽取 FIR (decimation) */
esp_err_t dsps_fird_init_f32(fir_f32_t *fir, float *coeffs, float *delay, int N, int decim);
int      dsps_fird_f32(fir_f32_t *fir, const float *input, float *output, int len);  /* 返回输出数 */

esp_err_t dsps_fird_init_s16(fir_s16_t *fir, int16_t *coeffs, int16_t *delay,
                             int16_t coeffs_len, int16_t decim, int16_t start_pos, int16_t shift);
int32_t  dsps_fird_s16(fir_s16_t *fir, const int16_t *input, int16_t *output, int32_t len);
esp_err_t dsps_fird_s16_aexx_free(fir_s16_t *fir);

/* 多速率 FIR (interp + decim) */
esp_err_t dsps_firmr_init_f32(fir_f32_t *fir, float *coeffs, float *delay,
                              int length, int interp, int decim, int start_pos);
int      dsps_firmr_f32(fir_f32_t *fir, const float *input, float *output, int input_len);
esp_err_t dsps_firmr_init_s16(fir_s16_t *fir, int16_t *coeffs, int16_t *delay,
                              int16_t length, int16_t interp, int16_t decim,
                              int16_t start_pos, int16_t shift);
int32_t  dsps_firmr_s16(fir_s16_t *fir, const int16_t *input, int16_t *output, int32_t input_len);

/* 辅助 */
esp_err_t dsps_16_array_rev(int16_t *arr, int16_t len);  /* 系数倒序（aes3 需要） */
```

## Resampler (`dsps_resampler.h`)

```c
typedef struct dsps_resample_mr_s {
    void *filter; float samplerate_factor; int16_t decim_c, decim_f, active_decim;
    float decim_avg_in, decim_avg_out;
    int32_t (*dsps_firmr)(void*, void*, void*, int32_t); int32_t fixed_point;
} dsps_resample_mr_t;

typedef struct dsps_resample_ph_s { float phase; float step; float delay[4]; int delay_pos; } dsps_resample_ph_t;

/* 多速率重采样器 */
esp_err_t dsps_resampler_mr_init(dsps_resample_mr_t *resampler, void *coeffs, int16_t length,
                                 int16_t interp, float samplerate_factor, int32_t fixed_point, int16_t shift);
int32_t  dsps_resampler_mr_exec(dsps_resample_mr_t *resampler, void *input, void *output,
                                int32_t length, int32_t length_correction);
void     dsps_resampler_mr_free(dsps_resample_mr_t *resampler);

/* 多项式（Farrow）重采样器 */
esp_err_t dsps_resampler_ph_init(dsps_resample_ph_t *resampler, float samplerate_factor);
int32_t  dsps_resampler_ph_exec(dsps_resample_ph_t *resampler, float *input, float *output, int32_t length);
```

## IIR Biquad (`dsps_biquad.h`, `dsps_biquad_gen.h`)

```c
/* 单声道 / 立体声滤波。coeffs[5]={b0,b1,b2,a1,a2}, a0=1; w[2] 或 w[4] */
esp_err_t dsps_biquad_f32(const float *input, float *output, int len, float *coef, float *w);
esp_err_t dsps_biquad_sf32(const float *input, float *output, int len, float *coef, float *w);

/* 系数生成器（freq 归一化 [0..0.5]） */
esp_err_t dsps_biquad_gen_lpf_f32(float *coeffs, float f, float q);
esp_err_t dsps_biquad_gen_hpf_f32(float *coeffs, float f, float q);
esp_err_t dsps_biquad_gen_bpf_f32(float *coeffs, float f, float q);
esp_err_t dsps_biquad_gen_bpf0db_f32(float *coeffs, float f, float q);
esp_err_t dsps_biquad_gen_notch_f32(float *coeffs, float f, float gain, float q);   /* gain: dB */
esp_err_t dsps_biquad_gen_allpass360_f32(float *coeffs, float f, float q);
esp_err_t dsps_biquad_gen_allpass180_f32(float *coeffs, float f, float q);
esp_err_t dsps_biquad_gen_peakingEQ_f32(float *coeffs, float f, float q);
esp_err_t dsps_biquad_gen_lowShelf_f32(float *coeffs, float f, float gain, float q);
esp_err_t dsps_biquad_gen_highShelf_f32(float *coeffs, float f, float gain, float q);
```

## 窗函数 (`dsps_wind.h`)

```c
void dsps_wind_hann_f32(float *window, int len);
void dsps_wind_blackman_f32(float *window, int len);            /* alpha=0.16 */
void dsps_wind_blackman_harris_f32(float *window, int len);
void dsps_wind_blackman_nuttall_f32(float *window, int len);
void dsps_wind_nuttall_f32(float *window, int len);
void dsps_wind_flat_top_f32(float *window, int len);
```

## 信号生成与测量 (`dsps_tone_gen.h`, `dsps_d_gen.h`, `dsps_h_gen.h`, `dsps_cplx_gen.h`, `dsps_snr.h`, `dsps_sfdr.h`)

```c
esp_err_t dsps_tone_gen_f32(float *output, int len, float Ampl, float freq, float phase); /* freq∈[-1..1] */
esp_err_t dsps_d_gen_f32(float *output, int len, int pos);   /* delta 函数 */
/* h_gen 头中提供 h（阶跃）生成器 dsps_h_gen_f32(...) */

typedef enum output_data_type { S16_FIXED = 0, F32_FLOAT = 1 } out_d_type;
typedef struct cplx_sig_s {
    void *lut; int32_t lut_len; float freq; float phase; out_d_type d_type; int16_t free_status;
} cplx_sig_t;
esp_err_t dsps_cplx_gen_init(cplx_sig_t *cplx_gen, out_d_type d_type, void *lut, int32_t lut_len,
                             float freq, float initial_phase);
esp_err_t dsps_cplx_gen(cplx_sig_t *cplx_gen, void *output, int32_t len);   /* output 长 len*2 */
esp_err_t dsps_cplx_gen_freq_set(cplx_sig_t *cplx_gen, float freq);
float    dsps_cplx_gen_freq_get(cplx_sig_t *cplx_gen);
esp_err_t dsps_cplx_gen_phase_set(cplx_sig_t *cplx_gen, float phase);
float    dsps_cplx_gen_phase_get(cplx_sig_t *cplx_gen);
esp_err_t dsps_cplx_gen_set(cplx_sig_t *cplx_gen, float freq, float phase);
void     cplx_gen_free(cplx_sig_t *cplx_gen);

float dsps_snr_f32(const float *input, int32_t len, uint8_t use_dc);   /* 单音 SNR(dB) */
float dsps_snr_fc32(const float *input, int32_t len);
float dsps_sfdr_f32(const float *input, int32_t len, int8_t use_dc);    /* SFDR(dB) */
float dsps_sfdr_fc32(const float *input, int32_t len);
```

## 向量数学 (`dsps_math.h` → add/sub/mul/addc/mulc/sqrt)

```c
/* out[i*step_out] = input1[i*step1] <op> input2[i*step2]; i=[0..len) */
esp_err_t dsps_add_f32 (const float *i1, const float *i2, float *out, int len, int step1, int step2, int step_out);
esp_err_t dsps_sub_f32 (const float *i1, const float *i2, float *out, int len, int step1, int step2, int step_out);
esp_err_t dsps_mul_f32 (const float *i1, const float *i2, float *out, int len, int step1, int step2, int step_out);
/* out[i*step_out] = input[i*step_in] + C / * C */
esp_err_t dsps_addc_f32(const float *input, float *output, int len, float C, int step_in, int step_out);
esp_err_t dsps_mulc_f32(const float *input, float *output, int len, float C, int step_in, int step_out);
/* 平方根近似 */
esp_err_t dsps_sqrt_f32(const float *input, float *output, int len);
float    dsps_sqrtf_f32_ansi(const float data);            /* 标量 */
float    dsps_inverted_sqrtf_f32_ansi(float data);         /* 1/sqrt(x) */

/* 定点（多 shift 参数） */
esp_err_t dsps_add_s16(const int16_t *i1, const int16_t *i2, int16_t *out, int len, int s1, int s2, int so, int shift);
esp_err_t dsps_sub_s16(const int16_t *i1, const int16_t *i2, int16_t *out, int len, int s1, int s2, int so, int shift);
esp_err_t dsps_mulc_s16(const int16_t *input, int16_t *output, int len, int16_t C, int step_in, int step_out);
esp_err_t dsps_add_s8 (const int8_t  *i1, const int8_t  *i2, int8_t  *out, int len, int s1, int s2, int so, int shift);
esp_err_t dsps_sub_s8 (const int8_t  *i1, const int8_t  *i2, int8_t  *out, int len, int s1, int s2, int so, int shift);
```

## 点积 (`dsps_dotprod.h`)

```c
esp_err_t dsps_dotprod_f32(const float *src1, const float *src2, float *dest, int len);
esp_err_t dsps_dotprode_f32(const float *src1, const float *src2, float *dest, int len, int step1, int step2);
esp_err_t dsps_dotprod_s16(const int16_t *src1, const int16_t *src2, int16_t *dest, int len, int8_t shift);
esp_err_t dsps_dp_s8(const int8_t *src1, const int8_t *src2, int32_t *dest, int len);  /* 结果 int32, 无 shift */
```

## 矩阵运算 (`dspm_matrix.h`, `dspm_mult.h`)

```c
esp_err_t dspm_mult_f32   (const float *A, const float *B, float *C, int m, int n, int k);  /* C[m][k]=A[m][n]*B[n][k] */
esp_err_t dspm_mult_ex_f32(const float *A, const float *B, float *C, int m, int n, int k, int A_padd, int B_padd, int C_padd);
esp_err_t dspm_mult_s16   (const int16_t *A, const int16_t *B, int16_t *C, int m, int n, int k, int shift);
/* 定长优化（无 shift 参数；未启用 ae32 时宏回退到通用 dspm_mult_f32） */
esp_err_t dspm_mult_3x3x1_f32(const float *A, const float *B, float *C);
esp_err_t dspm_mult_3x3x3_f32(const float *A, const float *B, float *C);
esp_err_t dspm_mult_4x4x1_f32(const float *A, const float *B, float *C);
esp_err_t dspm_mult_4x4x4_f32(const float *A, const float *B, float *C);
esp_err_t dspm_mult_mxn_1xm_int8(const int8_t *A, const int8_t *B, int32_t *C, int M, int N); /* C[1][M]=A[M][N]*B[1][N] */

/* 还有 dspm_add_f32 / dspm_sub_f32 / dspm_mulc_f32 / dspm_addc_f32（矩阵逐元素，签名见对应头） */
```

## 卷积 / 相关 / 2D 卷积 (`dsps_conv.h`, `dsps_corr.h`, `dspi_conv.h`)

```c
esp_err_t dsps_conv_f32(const float *Signal, int siglen, const float *Kernel, int kernlen, float *convout); /* 1D，输出长 siglen+kernlen-1 */
esp_err_t dsps_corr_f32(const float *Signal, int siglen, const float *Pattern, int patlen, float *dest);   /* siglen > patlen */
esp_err_t dspi_conv_f32(const image2d_t *in_image, const image2d_t *filter, image2d_t *out_image);          /* 2D 卷积 */
```

> `dspi_conv_f32` 行为（来自 `modules/conv/float/dspi_conv_f32_ansi.c`）：
> - 固定为 **`'same'` 模式**——函数内部执行 `out_image->size_x = in_image->size_x; out_image->size_y = in_image->size_y;`，即输出尺寸 = 输入尺寸（无 `'full'` 模式）。
> - 边缘处只用核内有效像素参与累加（不补零），核中心锚点为 `rest = (filter->size - 1) >> 1`。
> - 构造 `image2d_t` 时输出的 `size_x/size_y` 可填 0（函数会覆写），但 `stride_x`（行宽/row pitch，必须 ≥ 实际行元素数）、`step_x`/`step_y`（元素步长，常规为 1）必须正确。
> - 无论 `CONFIG_DSP_OPTIMIZED` 是否开启都映射到 `_ansi`，目前**没有** ae32/aes3/arp4 优化版本。
> - `image2d_t` 寻址：`Point[x,y] = data[stride_y*y*step_y + stride_x*x*step_x]`（`stride_x` 是行宽，不是子采样步长）。详见 `recipes/conv2d_image.md`。

## 视图 / 内存 (`dsps_view.h`, `dsps_mem.h`)

```c
void dsps_view(const float *data, int32_t len, int width, int height, float min, float max, char view_char);
void dsps_view_s16(const int16_t *data, int32_t len, int width, int height, float min, float max, char view_char);
void dsps_view_spectrum(const float *data, int32_t len, float min, float max);   /* 64x10 频谱 */

/* esp32s3 优化的 memcpy/memset（无优化时回退到标准 memcpy/memset） */
void *dsps_memcpy_aes3(void *arr_dest, const void *arr_src, size_t arr_len);
void *dsps_memset_aes3(void *arr_dest, uint8_t set_val, size_t set_size);
```

## C++ dspm::Mat 类 (`mat.h`)

构造：`Mat(int rows, int cols)` · `Mat(float *data, int rows, int cols)` · `Mat(float *data, int rows, int cols, int stride)` · `Mat()` · 拷贝构造。

成员：`rows, cols, stride, padding, data, length, abs_tol(static), ext_buff, sub_matrix`；内嵌 `struct Rect { x, y, width, height; ... }`。

| 方法 | 说明 |
|---|---|
| `operator()(row,col)` | 元素访问 |
| `operator=, +, -, *, /, +=, -=, *=, /=, ^=, ==` | 矩阵与常数运算 |
| `t()` | 转置 |
| `static Mat eye(int size)` / `ones(int size)` / `ones(int r,int c)` | 单位/全 1 阵 |
| `static Mat solve(Mat A, Mat b)` / `roots(Mat A, Mat y)` / `bandSolve(Mat A, Mat b, int k)` | 解方程 |
| `static Mat augment(Mat A, Mat B)` / `float dotProduct(Mat A, Mat B)` | 增广 / 点积 |
| `inverse()` / `pinv()` / `det(int n)` / `gaussianEliminate()` / `rowReduceFromGaussian()` | 逆/伪逆/行列式/高斯 |
| `swapRows(int,int)` / `clear()` / `normalize()` / `norm()` | 工具 |
| `getROI(...)` (3 重载) / `Get(...)` (2 重载) / `block(...)` / `Copy(src,row,col)` / `CopyHead(src)` / `PrintHead()` | 子矩阵/ROI |
| `operator<<` / `operator>>` | 流式 I/O |

## C++ EKF (`ekf.h`, `ekf_imu13states.h`)

```cpp
class ekf {                                  /* 基类 */
public:
    ekf(int x, int w);                       /* x=状态数, w=控制/噪声输入数 */
    virtual ~ekf();
    virtual void Process(float *u, float dt);/* 主处理：预测 */
    virtual void Init() = 0;
    int NUMX, NUMW;
    dspm::Mat &X;                            /* 状态向量 */
    /* F, G, ... 线性化矩阵（见头文件） */
    static dspm::Mat rotm2quat(dspm::Mat &R);
    static dspm::Mat quat2eul(const float q[4]);
    static dspm::Mat eul2rotm(float xyz[3]);
};

class ekf_imu13states : public ekf {         /* 13 态 IMU EKF */
public:
    ekf_imu13states();
    virtual void Init();
    virtual dspm::Mat StateXdot(dspm::Mat &x, float *u);
    virtual void LinearizeFG(dspm::Mat &x, float *u);
    dspm::Mat mag0, accel0;
    int NUMU;
    void UpdateRefMeasurement(float *accel_data, float *magn_data, float R[6]);               /* 常规运行 */
    void UpdateRefMeasurementMagn(float *accel_data, float *magn_data, float R[6]);           /* 标定阶段 */
    void UpdateRefMeasurement(float *accel_data, float *magn_data, float *attitude, float R[10]); /* 带参考姿态 */
    void Test(); void TestFull(bool enable_att);
};
```
