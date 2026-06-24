# 13 态 IMU 扩展卡尔曼滤波（EKF）

> **适用摘要**: 用 C++ 类 `ekf_imu13states`（继承自 `ekf`）做 IMU 姿态估计：初始化 → 标定阶段 → 每周期 `Process(gyro, dt)` → `UpdateRefMeasurement(accel, magn, R)` 修正陀螺仪偏差与姿态四元数。配合 `dspm::Mat` 与 `ekf` 静态方法（`eul2rotm`/`rotm2quat`/`quat2eul`）。

## 触发意图

- "卡尔曼滤波"
- "IMU 姿态估计"
- "EKF / 扩展卡尔曼"
- "陀螺仪偏差校正"
- "四元数姿态"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/kalman/main/ekf_imu13states_main.cpp` |
| 语言 | C++（`.cpp`），`app_main` 用 `extern "C"` |
| 头文件 | `esp_dsp.h` + `ekf_imu13states.h` |

## 分步说明

### 状态向量含义

`X` 为 13 维状态：
- `X[0..3]` — 姿态四元数
- `X[4..6]` — 陀螺仪偏差误差（rad/sec）
- `X[7..9]` — 磁力计向量幅值 `magn_ampl`
- `X[10..12]` — 磁力计偏移 `magn_offset`

参考磁力计值 = `magn_ampl * rotation_matrix' + magn_offset`。

### 最小调用骨架

```cpp
#include "esp_log.h"
#include "esp_dsp.h"
#include "ekf_imu13states.h"

extern "C" void app_main();

void app_main()
{
    ekf_imu13states *ekf13 = new ekf_imu13states();
    ekf13->Init();                 /* 必须先 Init */

    float dt = 0.01f;              /* 采样间隔（秒） */
    /* 测量噪声协方差（对角），越小越信任该测量 */
    float R[6] = {0.01f, 0.01f, 0.01f, 0.01f, 0.01f, 0.01f};

    while (running) {
        /* 1) 读取陀螺仪（rad/sec） */
        float u[3] = {gyro_x, gyro_y, gyro_z};
        /* 2) 预测：更新状态与协方差 */
        ekf13->Process(u, dt);
        /* 3) 读取加速度计（g，1g≈9.81m/s²）与磁力计，归一化后修正 */
        ekf13->UpdateRefMeasurement(accel_norm, magn_norm, R);

        /* 当前姿态在 ekf13->X.data[0..3]（四元数） */
    }
    delete ekf13;
}
```

### 两种修正方法

```cpp
/* 常规运行：仅更新姿态与陀螺仪偏差（标定后主用） */
ekf13->UpdateRefMeasurement(accel_data, magn_data, R /*float[6]*/);

/* 标定阶段：更新完整状态（含磁力计幅值/偏移） */
ekf13->UpdateRefMeasurementMagn(accel_data, magn_data, R /*float[6]*/);

/* 还可传入参考姿态四元数做完整修正（静止或初始化期） */
float attitude[4] = {...};
float R10[10] = {...};
ekf13->UpdateRefMeasurement(accel_data, magn_data, attitude, R10);
```

### 标定 → 运行 两阶段模式（官方示例）

```cpp
ekf_imu13states *ekf13 = new ekf_imu13states();
ekf13->Init();

/* 参考向量：重力方向、磁场方向 */
float accel0_data[] = {0, 0, 1};
float magn0_data[]  = {1, 0, 0};
dspm::Mat accel0(accel0_data, 3, 1);
dspm::Mat magn0(magn0_data, 3, 1);

/* ---- 标定阶段：用 UpdateRefMeasurementMagn 更新完整状态 ---- */
for (size_t n = 1; n < total_N * 16; n++) {
    /* emu 或真实陀螺数据 */
    float u[3] = {gyro_sample(0,0), gyro_sample(1,0), gyro_sample(2,0)};
    ekf13->Process(u, dt);
    ekf13->UpdateRefMeasurementMagn(accel_norm.data, magn_norm.data, R);
}
/* 标定结束后，应保存 ekf13->X 与协方差矩阵，下次开机恢复，跳过标定 */

/* ---- 运行阶段：用 UpdateRefMeasurement 只更新姿态+陀螺偏差 ---- */
for (size_t n = 1; n < total_N * 16; n++) {
    float u[3] = {gyro_sample(0,0), gyro_sample(1,0), gyro_sample(2,0)};
    ekf13->Process(u, dt);
    ekf13->UpdateRefMeasurement(accel_norm.data, magn_norm.data, R);
}

/* 提取结果 */
dspm::Mat estimated_error(&ekf13->X.data[4], 3, 1);   /* 陀螺偏差估计 */
dspm::Mat euler_deg = 180.0f / pi * ekf::quat2eul(ekf13->X.data);  /* 欧拉角（度） */
```

### 配套 ekf 静态方法（四元数/旋转矩阵/欧拉角互转）

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
| 崩溃 / 未初始化 | 漏 `Init()` | 构造后必须 `ekf13->Init()` |
| 姿态不收敛 | 跳过标定阶段 | 先用 `UpdateRefMeasurementMagn` 标定，再切 `UpdateRefMeasurement` |
| 单位错误 | 加速度未换算成 g | `accel` 单位为 g（1g≈9.81m/s²），陀螺为 rad/sec |
| `R` 数组长度错 | 用错重载 | 6 元素版用于 `UpdateRefMeasurement(accel,magn,R)`；10 元素版带 attitude |
| 开机需重新标定 | 未保存状态 | 标定后保存 `X` 与 `P`，下次恢复以节省时间（示例 README 明确建议） |
| `.c` 文件编译失败 | EKF 仅 C++ | 用 `.cpp`，`app_main` 加 `extern "C"` |

## 参考

- `examples/kalman/main/ekf_imu13states_main.cpp` — 完整标定+运行两阶段示例
- `examples/kalman/README.md` — 标定/状态保存恢复说明
- `modules/kalman/ekf/include/ekf.h` — 基类 `ekf`、`Process`、`Init`、静态方法 `rotm2quat`/`quat2eul`/`eul2rotm`
- `modules/kalman/ekf_imu13states/include/ekf_imu13states.h` — `UpdateRefMeasurement`/`UpdateRefMeasurementMagn`
