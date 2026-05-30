# Composite HID CDC Device

基于 STM32F103C8T6 的 USB 复合设备，实现 HID（键盘/鼠标）与 CDC（虚拟串口）功能。

## 功能特性

- **USB HID 键盘** - 支持全键盘输入、组合键
- **USB HID 鼠标** - 支持鼠标移动、点击、滚轮
- **USB CDC** - 虚拟串口通信
- **复合设备** - 单个USB设备同时支持HID和CDC

## 硬件要求

- STM32F103C8T6 开发板
- USB 接口

## 项目结构

```
├── Composite_CDC_HID/        # STM32 固件项目
│   ├── Core/                 # 核心代码
│   ├── Drivers/              # 驱动库
│   ├── ICode/                # 自定义代码
│   │   └── USB_HID_DRIVER/   # HID 驱动
│   ├── MDK-ARM/              # Keil 项目文件
│   └── USB_DEVICE/           # USB 设备配置
├── Hand/                     # Python 上位机
│   ├── hand.py               # 主程序
│   └── hand_config.py        # 配置文件
└── pc_yolov5_bridge/         # YOLOv5 自动瞄准桥接
    └── detect_and_send.py    # 目标检测与鼠标控制
```

## 快速开始

### 固件编译

1. 使用 Keil MDK 打开 `Composite_CDC_HID/MDK-ARM/Composite_CDC_HID.uvprojx`
2. 编译项目
3. 下载到 STM32F103C8T6

### Python 上位机

```bash
# 安装依赖
pip install pyserial

# 运行示例
cd Hand
python hand.py
```

## 通信协议

### 命令格式

所有命令通过串口发送，格式为 ASCII 字符串：

```
[时间标志][设备类型][数据...]
```

### 设备类型

| 类型 | 值 | 说明 |
|------|-----|------|
| 键盘 | `1` | 9字节报告 |
| 多媒体 | `2` | 2字节报告 |
| 鼠标 | `3` | 5字节报告 |

### 时间标志

| 标志 | 值 | 说明 |
|------|-----|------|
| 点击 | `0` | 按下后自动释放 |
| 按住 | `1` | 保持按下状态 |

### 键盘命令

格式：`[时间标志][1][修饰键][保留][按键1-6]`

**示例：**

| 功能 | 命令 |
|------|------|
| 点击 'a' | `010000040000000000` |
| 点击 'b' | `010005000000000000` |
| Ctrl+A | `010104000000000000` |

### 鼠标命令

格式：`[时间标志][3][按键][X][Y][滚轮]`

**示例：**

| 功能 | 命令 |
|------|------|
| 左键点击 | `0301000000` |
| 右键点击 | `0302000000` |
| 向右移动10 | `03000A0000` |
| 向下移动50 | `0300003200` |

## Bug 修复记录

### PMA 缓冲区重叠 (Code 10)

**问题描述：** Windows 报告 Code 10 错误，USB 设备无法枚举。

**问题分析：**

这次 Code 10 更像是 USB PMA 端点缓冲区分配冲突导致的枚举失败，不像是 MCU heap 或总 RAM 不够。

检查后发现当前 PMA 分配里，CDC OUT 和 HID IN 的地址区间发生了重叠，这是一个实质性错误。

**关键问题：**

在 `usbd_conf.c` 中，当前分配是：
- `CDC_OUT_EP` 起始 `0x110`
- `CUSTOM_HID_EPIN_ADDR` 起始 `0x140`

根据 `usbd_cdc.h`，`CDC_OUT_EP` 是 FS Bulk，包长 64 字节，所以它占用区间是 `0x110 ~ 0x14F`。

根据 `usbd_customhid.h`，`CUSTOM_HID_EPIN_SIZE = 0x0D`，HID IN 从 `0x140` 开始，至少占到 `0x14C`。

所以这两段明显重叠：`0x140 ~ 0x14C`。这种冲突足以让主机枚举异常，Windows 就可能报"系统资源不够，无法完成 API / 代码10"。

**修复方案：** 重新规划 PMA 分配：

| 端点 | PMA地址 | 占用区间 |
|------|---------|----------|
| EP0_OUT | 0x00 | 0x00-0x3F |
| EP0_IN | 0x40 | 0x40-0x7F |
| CDC_IN | 0x80 | 0x80-0xBF |
| CDC_OUT | 0xC0 | 0xC0-0xFF |
| CDC_CMD | 0x100 | 0x100-0x107 |
| HID_IN | 0x108 | 0x108-0x115 |
| HID_OUT | 0x116 | 0x116-0x123 |

## 开发环境

- STM32CubeMX
- Keil MDK-ARM
- Python 3.8+
- pyserial

## 许可证

MIT License

## 参考

- [cnxft/composite_hid_cdc](https://github.com/cnxft/composite_hid_cdc)
- [STM32 USB Device Library](https://www.st.com/en/embedded-software/stm32-usb-device-library.html)
- [USB HID Specification](https://www.usb.org/hid)
