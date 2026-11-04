# ESP32-CAM_MJPEG2SD 项目说明文档

## 项目概述

ESP32-CAM_MJPEG2SD 是一个用于ESP32/ESP32S3开发板的应用程序，主要功能是：
- 从OV2640/OV3660/OV5640/PY260摄像头捕获JPEG图像
- 将图像存储为AVI文件到SD卡
- 支持通过浏览器以MJPEG流形式播放AVI文件
- 支持音频录制（如果安装了麦克风）
- 支持运动检测、连续录制、时间间隔拍摄等功能

## 主要功能模块

### 1. 核心录制功能 (`mjpeg2sd.cpp`)
- `prepRecording()` - 初始化录制任务
- `startRecording()` - 开始录制AVI文件
- `stopRecording()` - 停止录制
- `saveFrame()` - 保存单帧图像

### 2. SD卡存储管理 (`utilsFS.cpp`)
- `prepSD_MMC()` - 初始化SD卡
- `startStorage()` - 启动存储系统
- `getFreeStorage()` - 获取存储空间信息
- `checkFreeSpace()` - 检查可用空间

### 3. 摄像头控制 (`appSpecific.cpp`)
- `prepCam()` - 初始化摄像头
- `updateFrame()` - 更新摄像头帧
- `setCamParams()` - 设置摄像头参数

### 4. 网络服务 (`webServer.cpp`)
- `startWebServer()` - 启动Web服务器
- `streamServer()` - 流媒体服务器
- `handleWebRequests()` - 处理HTTP请求

### 5. 运动检测 (`motionDetect.cpp`)
- `motionDetect()` - 运动检测算法
- `checkMotion()` - 检查运动状态

## 关键函数和配置注意事项

### 对于XIAO_esp32s3读取SD卡失败的问题

#### 1. SD卡引脚配置 (`camera_pins.h:247-249`)
```cpp
// XIAO_ESP32S3的SD卡引脚定义
#define SD_MMC_CLK 7  // 时钟引脚
#define SD_MMC_CMD 9  // 命令引脚
#define SD_MMC_D0  8  // 数据引脚0
```

**重要提示**：如果SD卡读取失败，请检查以下配置：

#### 2. SD卡初始化函数 (`utilsFS.cpp:55-88`)
- `prepSD_MMC()` 函数负责初始化SD卡
- 需要正确设置引脚模式（1位或4位模式）
- 对于XIAO_esp32s3，默认使用1位模式

#### 3. 必须正确设置的函数：

##### a) 引脚设置函数 (`utilsFS.cpp:66-73`)
```cpp
// 对于ESP32S3，需要正确设置引脚
#if CONFIG_IDF_TARGET_ESP32S3
#if defined(SD_MMC_D1)
    // 4位模式
    SD_MMC.setPins(SD_MMC_CLK, SD_MMC_CMD, SD_MMC_D0, SD_MMC_D1, SD_MMC_D2, SD_MMC_D3);
    use1bitMode = false;
#else
    // 1位模式（XIAO_esp32s3默认）
    SD_MMC.setPins(SD_MMC_CLK, SD_MMC_CMD, SD_MMC_D0);
#endif
#endif
```

##### b) SD卡挂载函数 (`utilsFS.cpp:77`)
```cpp
res = SD_MMC.begin("/sdcard", use1bitMode, formatIfMountFailed, sdmmcFreq);
```
- `use1bitMode`: XIAO_esp32s3应设置为true（1位模式）
- `formatIfMountFailed`: 挂载失败时是否格式化（建议设置为false进行调试）
- `sdmmcFreq`: SD卡时钟频率（默认20MHz）

#### 4. 可能导致SD卡读取失败的原因：

1. **引脚配置错误**：
   - 检查`camera_pins.h`中XIAO_ESP32S3的SD卡引脚定义是否正确
   - 确认实际硬件连接与代码定义一致

2. **SD卡模式设置错误**：
   - XIAO_esp32s3应使用1位模式（`use1bitMode = true`）
   - 如果错误地启用了4位模式，会导致引脚冲突

3. **时钟频率问题**：
   - 某些SD卡可能不支持高速模式
   - 可以尝试降低`sdmmcFreq`值

4. **电源问题**：
   - SD卡需要稳定的3.3V电源
   - XIAO_esp32s3的电源可能不足，考虑外接电源

5. **SD卡格式问题**：
   - 确保SD卡格式化为FAT32格式
   - 建议使用官方工具格式化

## 项目配置文件

### `appGlobals.h` - 主要配置文件

#### 开发板选择（第33行）：
```cpp
#define CAMERA_MODEL_XIAO_ESP32S3
```
**必须取消注释XIAO_ESP32S3，并确保只启用一个开发板定义**

#### SD卡相关配置：
- `formatIfMountFailed` - 挂载失败时自动格式化（建议调试时设为false）
- `sdMinCardFreeSpace` - 最小可用空间阈值（MB）
- `sdFreeSpaceMode` - 空间不足时的处理模式

### `camera_pins.h` - 引脚配置文件

XIAO_esp32s3的完整引脚配置：
- 摄像头引脚：GPIO10（XCLK）、GPIO40（SIOD）、GPIO39（SIOC）等
- SD卡引脚：GPIO7（CLK）、GPIO9（CMD）、GPIO8（D0）
- LED引脚：GPIO21

## 调试建议

如果XIAO_esp32s3读取SD卡失败：

1. **检查日志输出**：
   - 查看串口日志中的SD卡初始化信息
   - 注意是否有"SD card mount failed"错误

2. **验证硬件连接**：
   - 使用万用表检查SD卡引脚连接
   - 确认SD卡座接触良好

3. **测试SD卡**：
   - 在电脑上测试SD卡是否正常工作
   - 尝试不同的SD卡（建议使用Class 10或更高）

4. **修改配置**：
   - 将`formatIfMountFailed`设置为false
   - 降低`sdmmcFreq`频率（如10MHz）
   - 检查`use1bitMode`设置

5. **电源检查**：
   - 测量SD卡供电电压（应为3.3V）
   - 考虑使用外部电源或电容稳压

## 项目结构

```
ESP32-CAM_MJPEG2SD/
├── ESP32-CAM_MJPEG2SD.ino    # 主程序入口
├── appGlobals.h              # 全局配置和宏定义
├── camera_pins.h             # 各开发板引脚定义
├── mjpeg2sd.cpp              # 核心录制功能
├── utilsFS.cpp               # SD卡和文件系统工具
├── webServer.cpp             # Web服务器
├── motionDetect.cpp          # 运动检测
├── audio.cpp                 # 音频功能
├── avi.cpp                   # AVI文件处理
├── peripherals.cpp           # 外设控制
└── data/                     # Web界面文件
```

## 总结

ESP32-CAM_MJPEG2SD是一个功能丰富的摄像头录制项目，特别适合安防监控、野生动物观察等应用。对于XIAO_esp32s3用户，需要特别注意SD卡引脚配置和初始化设置，确保正确使用1位模式并检查硬件连接。

**关键要点**：
1. 正确选择开发板定义（`CAMERA_MODEL_XIAO_ESP32S3`）
2. 验证SD卡引脚配置与实际硬件一致
3. 使用1位SD卡模式（`use1bitMode = true`）
4. 确保电源稳定，SD卡格式正确
5. 通过串口日志调试SD卡初始化问题