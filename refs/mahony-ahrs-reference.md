# Mahony AHRS 姿态解算算法参考

> 本文件作为 IMU/陀螺仪开发的高级参考，提供 Mahony AHRS（Attitude and Heading Reference System）算法的完整实现和调优指南。
>
> **适用场景**：需要高精度姿态解算的应用（平衡车、无人机、机器人等），特别是需要避免欧拉角万向锁问题的场景。

---

## 算法概述

### 核心思想

Mahony AHRS 算法是一个基于**四元数**的非线性互补滤波器，通过融合陀螺仪（角速度）和加速度计（重力方向）数据，实时计算系统的姿态。

### 与互补滤波的对比

| 特性 | 互补滤波 | Mahony AHRS |
|------|---------|-------------|
| 表示方式 | 欧拉角 | 四元数 |
| 万向锁 | 存在 | 无 |
| 计算量 | 极小 | 中等 |
| 精度 | 中等 | 高 |
| 适用场景 | 简单倾斜检测 | 3D 空间姿态解算 |

---

## 完整代码实现

### 1. 数据结构定义

```c
#include <math.h>

#ifndef PI
#define PI 3.141592654
#endif

// 角度弧度转换
#define Ang2Rad 0.01745329252f  // 角度 → 弧度
#define Rad2Ang 57.295779513f   // 弧度 → 角度

// 算法参数
#define sampleFreq  200.0f       // 采样频率（Hz）
#define twoKpDef    (2.0f * 0.5f)  // 比例增益 = 1.0
#define twoKiDef    (2.0f * 0.0f)  // 积分增益 = 0.0（可禁用）
```

### 2. 全局变量

```c
// 四元数（表示姿态）
volatile float q0 = 1.0f, q1 = 0.0f, q2 = 0.0f, q3 = 0.0f;

// 积分误差项
volatile float integralFBx = 0.0f, integralFBy = 0.0f, integralFBz = 0.0f;

// 算法参数（可运行时调整）
volatile float invsampleFreq = 1.0f / sampleFreq;
volatile float twoKp = twoKpDef;
volatile float twoKi = twoKiDef;

// 输出姿态角（弧度）
float Pitch_a_Pi, Roll_a_Pi, Yaw_a_Pi;
```

### 3. 核心算法函数

```c
void MahonyAHRSupdateIMU(float gx, float gy, float gz,
                         float ax, float ay, float az)
{
    // 静态变量（优化：避免重复分配）
    static float recipNorm;
    static float halfvx, halfvy, halfvz;
    static float halfex, halfey, halfez;
    static float qa, qb, qc;

    // 仅在有加速度时计算（静止检测）
    if(!((ax == 0.0f) && (ay == 0.0f) && (az == 0.0f)))
    {
        // 1. 角速度单位转换：°/s → rad/s
        gx *= 0.0174532925f;
        gy *= 0.0174532925f;
        gz *= 0.0174532925f;

        // 2. 归一化加速度计测量值
        recipNorm = invSqrt(ax * ax + ay * ay + az * az);
        ax *= recipNorm;
        ay *= recipNorm;
        az *= recipNorm;

        // 3. 从四元数估计重力方向（旋转矩阵的前三行第三列）
        halfvx = q1 * q3 - q0 * q2;
        halfvy = q0 * q1 + q2 * q3;
        halfvz = q0 * q0 - 0.5f + q3 * q3;

        // 4. 计算误差：估计重力与测量重力的交叉积
        halfex = (ay * halfvz - az * halfvy);
        halfey = (az * halfvx - ax * halfvz);
        halfez = (ax * halfvy - ay * halfvx);

        // 5. 积分反馈（修正陀螺仪长期漂移）
        if(twoKi > 0.0f)
        {
            integralFBx += twoKi * halfex * invsampleFreq;
            integralFBy += twoKi * halfey * invsampleFreq;
            integralFBz += twoKi * halfez * invsampleFreq;
            gx += integralFBx;
            gy += integralFBy;
            gz += integralFBz;
        }
        else {
            integralFBx = 0.0f;  // 防止积分饱和
            integralFBy = 0.0f;
            integralFBz = 0.0f;
        }

        // 6. 比例反馈（快速修正）
        gx += twoKp * halfex;
        gy += twoKp * halfey;
        gz += twoKp * halfez;
    }

    // 7. 四元数微分方程积分
    gx *= (0.5f * invsampleFreq);
    gy *= (0.5f * invsampleFreq);
    gz *= (0.5f * invsampleFreq);

    qa = q0;
    qb = q1;
    qc = q2;
    q0 += (-qb * gx - qc * gy - q3 * gz);
    q1 += (qa * gx + qc * gz - q3 * gy);
    q2 += (qa * gy - qb * gz + q3 * gx);
    q3 += (qa * gz + qb * gy - qc * gx);

    // 8. 四元数归一化
    recipNorm = invSqrt(q0 * q0 + q1 * q1 + q2 * q2 + q3 * q3);
    q0 *= recipNorm;
    q1 *= recipNorm;
    q2 *= recipNorm;
    q3 *= recipNorm;

    // 9. 从四元数计算欧拉角
    static float r11, r12, r21, r31, r32;
    r11 = 2.0f * (q0 * q1 + q2 * q3);
    r12 = 1.0f - 2.0f * (q1 * q1 + q2 * q2);
    r21 = 2.0f * (q0 * q2 - q3 * q1);
    r31 = 2.0f * (q0 * q3 + q1 * q2);
    r32 = 1.0f - 2.0f * (q2 * q2 + q3 * q3);

    // Z-Y-X 欧拉角（弧度）
    Yaw_a_Pi   = atan2f(r31, r32);   // 偏航角
    Pitch_a_Pi = -asinf(r21);        // 俯仰角（注意负号）
    Roll_a_Pi  = atan2f(r11, r12);   // 横滚角
}
```

### 4. 快速平方根倒数（优化）

```c
// 快速计算 1/sqrt(x)，使用 Quake III 算法
float invSqrt(float x)
{
    float halfx = 0.5f * x;
    float y = x;
    long i = *(long*)&y;
    i = 0x5f3759df - (i >> 1);
    y = *(float*)&i;
    y = y * (1.5f - (halfx * y * y));
    return y;
}
```

---

## 信号预处理（二阶滤波器）

在进入 Mahony 算法前，建议对原始传感器数据进行滤波，提高稳定性。

### 滤波器类型定义

```c
typedef enum
{
    BIQUAD_LOWPASS,        // 低通滤波器
    BIQUAD_HIGHPASS,       // 高通滤波器
    BIQUAD_BANDPASS_PEAK,  // 带通滤波器（峰值）
    BIQUAD_BANDSTOP_NOTCH, // 带阻滤波器（陷波）
} biquad_type;

typedef struct
{
    float a0, a1, a2, a3, a4;  // 滤波器系数
    float x1, x2, y1, y2;      // 状态变量
} biquad_state;
```

### 滤波器初始化

```c
void biquad_filter_init(biquad_state *state, biquad_type type,
                         int fs, float fc, float q_value)
{
    float w0 = 2 * PI * fc / fs;
    float sin_w0 = sinf(w0);
    float cos_w0 = cosf(w0);
    float alpha = sin_w0 / (2.0f * q_value);

    float b0, b1, b2, a0, a1, a2;

    switch(type)
    {
    case BIQUAD_LOWPASS:
        b0 = (1.0 - cos_w0) / 2.0;
        b1 = b0 * 2;
        b2 = b0;
        a0 = 1.0 + alpha;
        a1 = -2.0 * cos_w0;
        a2 = 1.0 - alpha;
        break;
    case BIQUAD_HIGHPASS:
        b0 = (1.0 + cos_w0) / 2.0;
        b1 = -b0 * 2;
        b2 = b0;
        a0 = 1.0 + alpha;
        a1 = -2.0 * cos_w0;
        a2 = 1.0 - alpha;
        break;
    case BIQUAD_BANDPASS_PEAK:
        b0 = alpha;
        b1 = 0.0;
        b2 = -alpha;
        a0 = 1.0 + alpha;
        a1 = -2.0 * cos_w0;
        a2 = 1.0 - alpha;
        break;
    case BIQUAD_BANDSTOP_NOTCH:
        b0 = 1.0;
        b1 = -2.0 * cos_w0;
        b2 = 1.0;
        a0 = 1.0 + alpha;
        a1 = -2.0 * cos_w0;
        a2 = 1.0 - alpha;
        break;
    }

    state->a0 = b0 / a0;
    state->a1 = b1 / a0;
    state->a2 = b2 / a0;
    state->a3 = a1 / a0;
    state->a4 = a2 / a0;
    state->x1 = state->x2 = 0.0;
    state->y1 = state->y2 = 0.0;
}
```

### 滤波器处理函数

```c
float biquad(biquad_state *state, float data)
{
    float result = state->a0 * data + state->a1 * state->x1 +
                  state->a2 * state->x2 - state->a3 * state->y1 -
                  state->a4 * state->y2;

    state->x2 = state->x1;
    state->x1 = data;
    state->y2 = state->y1;
    state->y1 = result;

    return result;
}
```

### 典型滤波器配置

```c
// 采样率 200Hz 时的推荐配置
void Filter_Init(void)
{
    // 角速度滤波
    biquad_filter_init(&Pitch_g_biquad, BIQUAD_LOWPASS, 200, 30, 0.7071);
    biquad_filter_init(&Roll_g_biquad,  BIQUAD_LOWPASS, 200, 20, 0.5773);
    biquad_filter_init(&Yaw_g_biquad,   BIQUAD_LOWPASS, 200, 30, 0.5773);

    // 加速度计滤波
    biquad_filter_init(&Pitch_acc_biquad, BIQUAD_LOWPASS, 200, 10, 0.5773);
    biquad_filter_init(&Roll_acc_biquad,  BIQUAD_LOWPASS, 200, 10, 0.5773);
    biquad_filter_init(&Yaw_acc_biquad,   BIQUAD_LOWPASS, 200, 10, 0.5773);
}
```

### Q 值选择参考

| 滤波器类型 | Q 值 | 特点 | 适用场景 |
|-----------|------|------|---------|
| Butterworth | 0.7071 | 最平坦的幅频响应，最小相位失真 | 一般应用 |
| Bessel | 0.5773 | 良好的时域响应，低延迟 | 实时控制系统 |
| Chebyshev (1dB) | 0.9565 | 阻带衰减快，但有通带波纹 | 需要强滤波时 |

---

## 完整使用示例

```c
// 滤波器状态
biquad_state Pitch_g_biquad, Roll_g_biquad, Yaw_g_biquad,
             Pitch_acc_biquad, Roll_acc_biquad, Yaw_acc_biquad;

// 原始数据
float Pitch_acc, Roll_acc, Yaw_acc;
float Pitch_g, Roll_g, Yaw_g;

// 滤波后数据
float Pitch_acc_F, Roll_acc_F, Yaw_acc_F;
float Pitch_g_F, Roll_g_F, Yaw_g_F;

// 姿态角（角度）
float Pitch_a, Roll_a, Yaw_a;

// Yaw 角多圈计数
float Yaw_TotalAngle = 0.0f;
float Yaw_AngleLast = 0.0f;
int Yaw_RoundCount = 0;

void get_IMU_data(void)
{
    // 1. 获取原始数据（假设已实现）
    Get_Acc_ICM42688();
    Get_Gyro_ICM42688();

    // 2. 轴映射（根据实际硬件调整）
    Pitch_acc = -icm42688_acc_y;
    Roll_acc  =  icm42688_acc_x;
    Yaw_acc   =  icm42688_acc_z;

    Pitch_g = -icm42688_gyro_y;
    Roll_g  =  icm42688_gyro_x;
    Yaw_g   =  icm42688_gyro_z;

    // 3. 滤波
    Pitch_acc_F = biquad(&Pitch_acc_biquad, Pitch_acc);
    Roll_acc_F  = biquad(&Roll_acc_biquad, Roll_acc);
    Yaw_acc_F   = biquad(&Yaw_acc_biquad, Yaw_acc);

    Pitch_g_F = biquad(&Pitch_g_biquad, Pitch_g);
    Roll_g_F  = biquad(&Roll_g_biquad, Roll_g);
    Yaw_g_F   = biquad(&Yaw_g_biquad, Yaw_g);

    // 4. Mahony AHRS 解算
    MahonyAHRSupdateIMU(icm42688_gyro_x, icm42688_gyro_y, icm42688_gyro_z,
                        icm42688_acc_x, icm42688_acc_y, icm42688_acc_z);

    // 5. 弧度转角度
    Pitch_a  = -Rad2Ang * Pitch_a_Pi;
    Roll_a   =  Rad2Ang * Roll_a_Pi;
    Yaw_a    =  Rad2Ang * Yaw_a_Pi;

    // 6. Yaw 角多圈计数处理
    if (Yaw_a - Yaw_AngleLast > 180.0f) {
        Yaw_RoundCount--;
    } else if (Yaw_a - Yaw_AngleLast < -180.0f) {
        Yaw_RoundCount++;
    }
    Yaw_TotalAngle = 360.0f * Yaw_RoundCount + Yaw_a;
    Yaw_AngleLast = Yaw_a;
}
```

---

## 参数调优指南

### 1. Kp（比例增益）调优

**作用**：控制加速度计修正陀螺仪漂移的响应速度。

| Kp 值 | 效果 | 适用场景 |
|-------|------|---------|
| 0.5 - 1.0 | 修正慢，陀螺仪权重高 | 快速运动、振动大的场合 |
| 1.0 - 2.0 | 响应适中（推荐） | 通用场景 |
| 2.0 - 5.0 | 修正快，加速度计权重高 | 静止或缓慢运动 |

**调优方法**：
- 静止时，观察角度是否快速收敛到正确值
- 快速倾斜时，观察角度是否有明显延迟或振荡

### 2. Ki（积分增益）调优

**作用**：修正陀螺仪的长期漂移（零偏）。

| Ki 值 | 效果 | 适用场景 |
|-------|------|---------|
| 0.0 | 禁用积分 | 已校准陀螺仪零偏 |
| 0.01 - 0.1 | 弱积分 | 零偏较小 |
| 0.1 - 0.5 | 强积分 | 零偏较大 |

**调优方法**：
- 静置 10 分钟，观察角度漂移是否逐渐减小
- 积分过大会导致动态响应变慢

### 3. 采样频率

**推荐**：100Hz - 500Hz

```c
// 采样频率越高，解算精度越高，但 CPU 负载越大
#define sampleFreq  200.0f  // 平衡车典型值
```

---

## 常见问题排查

### 1. 角度发散
- 检查轴映射是否正确
- 检查采样频率是否与实际匹配
- 适当增大 Kp 值

### 2. 角度漂移
- 陀螺仪未校准，启用 Ki 或先校准零偏
- 检查加速度计是否受振动影响，降低截止频率

### 3. 角度振荡
- Kp 值过大，适当减小
- 检查加速度计噪声，增强滤波

### 4. Yaw 角不准
- Mahony AHRS（无磁力计）只能准确估计 Pitch 和 Roll
- Yaw 角需配合磁力计或视觉传感器

---

## 算法优缺点总结

### ✅ 优点
- **无万向锁**：四元数表示，全姿态覆盖
- **计算效率高**：相比 EKF，计算量小，适合嵌入式
- **稳定性好**：长期漂移可控
- **易于调优**：参数物理意义明确

### ⚠️ 缺点
- **Yaw 角不可靠**：仅靠加速度计无法估计偏航
- **对振动敏感**：加速度计受振动影响时精度下降
- **需预滤波**：原始数据噪声需滤波处理

---

## 扩展参考

### 相关算法对比
- **Madgwick AHRS**：计算稍复杂，收敛更快
- **扩展卡尔曼滤波（EKF）**：精度最高，计算量最大
- **互补滤波**：最简单，仅适用于 2D 场景

### 进阶优化
- 自适应 Kp/Ki 参数
- 陀螺仪零偏在线校准
- 多传感器融合（磁力计、气压计、GPS）

---

## 完整工程示例

本参考实现来自 **TI MSPM0G3507 逐飞开源库**项目：
- 传感器：ICM-42688
- 采样率：200Hz
- 滤波器：二阶 Biquad 低通
- 应用场景：平衡车/机器人姿态解算

> 本文档代码可直接移植到 STM32/ESP32/Arduino 等平台，仅需替换底层数据读取函数。
